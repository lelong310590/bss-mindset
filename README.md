## Task 555
### Overview:
>**Main task:** thay đổi flow từ sử dụng [redux-saga]() sang sử dụng [redux-toolkit]().

>**Sub task:** refactor code

### Report:
Trong quá trình thực hiện thay đổi từ việc chuyển đổi thư viện state manager từ redux-saga sang redux toolkit thì có thực hiện refactor source dẫn đến một vài mindset và flow khi triển khai sẽ thay đổi theo. 
Sau đây sẽ là report lại việc thay đổi đó

### I. Theo Flow & Mind set cũ

Dựa trên dự án đang tham gia là **B2B-Customer-Portal** , nhân thấy để triển khai 1 UI module manager theo flow và mindset cũ thì sẽ có các bước như sau (exp: module: "qo")

**1. Tạo page tương ứng (index, create, edit)**

Tại page này sẽ bao gồm 3 phần

>a. SSR

example code: 

<pre>
export const getServerSideProps = wrapper.getServerSideProps (store => async ({req, res}) => {
    
    let allCookies = cookies({ req })

    let domain = allCookies.bssBCPShopOrigin

    let token = allCookies.bssBCPAccessToken

    if (!domain || !token) {
        res.writeHead(302, {
            Location: '/login.html'
        });
        res.end();
        return;
    }

    if (!store.getState().qoRules.rules.length) {;
        store.dispatch({
            type: 'GET_QO_RULES_ASYNC',
            domain: domain
        })
    }

    if (!store.getState().general.user.shopifyPlan) {
        store.dispatch({
            type: 'QO_GET_SHOP_PLAN_ASYNC',
            domain: domain,
        })
    }


    store.dispatch(END);
        await store.sagaTask.toPromise();

        return {props: {domain: domain}};
    }
)
</pre>

Đa số tất cả các module đều có một số bước lặp đi lặp lại là: lấy `domain` & `token` hoặc lấy `id` từ cookie và router rồi sau đó thực hiện các step sau
<ul>
    <li>Check authen bằng cách kiểm tra xem có tồn tại <code>domain</code> hoặc <code>token</code> bất kỳ nào đó không ? Nếu có thì sẽ cho next, không thì redirect 302 về trang login </li>
    <li>Lấy thông tin của shop sau đó thực hiện <code>dispatch</code> đến <code>reducer</code> để lưu thông tin này vào <code>state</code> </li>
    <li>Lấy setting của module theo domain rồi dispatch để lưu state</li>
    <li>Lấy thông tin của 1 module unit rồi rồi dispatch để lưu state</li>
</ul>

**2. Tại saga :**

Create 3 file (example module qo) để đóng vai trò điều phối (dispatcher) dựa trên các action và reducer với các func như sau:

<ul>
    <li>CRUD 1 module unit </li>
    <li>Lấy các tài nguyên liên quan thông qua api cho việc xử lý logic</li>
</ul>

Tất cả các `http request` trong saga đều ở trong các func `generator` phục vụ cho `coroutines`. Theo các lặp lại là:

<ol>
    <li>Khởi 1 biến tương ứng với request</li>
    <li>khởi tạo tiếp 1 biến nữa tương ứng với parse Json từ response trả về cho request trên</li>
</ol>

example code: 

<pre>
import { call, put, all, takeEvery, takeLatest } from 'redux-saga/effects';
import {
    QO_SAVE_RULE_ASYNC,
    QO_SAVE_RULE_SUCCESS,
    QO_SAVE_RULE_FAIL ,
    GET_RULE_BY_ID_SUCCESS,
    GET_RULE_BY_ID_ASYNC,
    GET_RULE_BY_ID_FAIL
} from "../../actions";
import { reduxGetDataLog, reduxSaveDataLog } from 'log'
const saveQoRuleToServer = (domain, rule) => fetch(`${API_URL}/qo-rule/save`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ domain: domain, rule: rule})
});

const getRuleById = (domain, id) => fetch(`${API_URL}/qo-rule/get-by-id`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ domain: domain, id: id})
});

function* saveQoRule(action) {
    =========== some logic =========
}

function* getRule(action) {
    try {
        const [getRuleByIdReq] = yield all([
            getRuleById(action.domain, action.id)
        ])
        const [getRuleByIdRes] = yield all([
            getRuleByIdReq.json()
        ])
        if (getRuleByIdRes.success) {
            yield put({
                type: GET_RULE_BY_ID_SUCCESS,
                rule: getRuleByIdRes.rule
            });
        } else {
            reduxGetDataLog(action.domain, 'FAILED', getRuleByIdRes, 'Failed get QO_RULE Location: redux/sagas/qo/qoRule.js line 76')
            yield put({
                type: GET_RULE_BY_ID_FAIL
            });
        }

    } catch (e) {
        reduxGetDataLog(action.domain, 'FAILED', e, 'Failed get QO_RULE Location: redux/sagas/qo/qoRule.js line 83')
        yield put({type: GET_RULE_BY_ID_FAIL, message: e.message});
    }
}
export default function* handleRuleData() {
    yield takeLatest(QO_SAVE_RULE_ASYNC, saveQoRule);O
    yield takeLatest(GET_RULE_BY_ID_ASYNC, getRule);
}
</pre>

**3 Các bước xử lý call dispatch và action phía page và components ...**
Xử lý các logic để call dispatch và action , change state, ......


🚩🚩🚩🚩🚩🚩🚩
> Các vấn đề hiện tại của source theo mindset cũ (không đề cập đến việc thay đổi state manager lib) là :

<ul>
    <li>Có sự lặp lại ở các func request http ( các thành phần về method, headers, body được khai báo lặp lại nhiều)</li>
    <li>Không quản lý được các endpoint. Trong trường hợp thay đổi endpoint thì sẽ phải thay đổi tại rất nhiều nơi</li>
    <li>Việc call các resource api, các config api, planshop api bị lặp lại nhiều</li>
    <li>.......</li>
</ul>

⭐⭐⭐⭐⭐⭐⭐

### II. Đề xuất nhỏ cho việc refactor

**1. quản lý endpoint tập trung**

example http.js

<pre>
export const GET_QO_RULE_BY_DOMAIN = `${API_URL}/qo-rule/get-by-domain`
export const GET_QO_RULE_BY_ID = `${API_URL}/qo-rule/get-by-id`
export const POST_QO_RULE = `${API_URL}/qo-rule/save`
export const GET_CONFIGURATION = `${API_URL}/configurations/get`
export const SAVE_CONFIGURATION = `${API_URL}/configurations/save`
export const GET_PRODUCT_COLLECTION = `${API_URL}/product/collections`
export const GET_CUSTOMER_TAGS = `${API_URL}/customer/tags`
export const GET_PRODUCT_TAGS = `${API_URL}/product/tags`
export const GET_PRODUCT_LIST = `${API_URL}/product/list`
export const GET_CUSTOMERS = `${API_URL}/customer/customers`
export const GET_CUSTOMERS_LIST_BY_ID = `${API_URL}/customer/list-by-ids`
export const GET_PRODUCT_LIST_BY_ID = `${API_URL}/product/list-by-ids`
export const UPDATE_SHOP_MODULES = `${API_URL}/module/update-shop-modules`
export const GET_SHOP = `${API_URL}/shop/get`
export const UPDATE_SHOPIFY_PLAN = `${API_URL}/shop/update-shopify-plan`
export const GET_MAIN_THEME_ID = `${API_URL}/shop/get-main-theme-id`
export const SHOP_UPLOAD_CONTENT = `${API_URL}/shop/upload-content`
export const GET_RO_RULES_BY_DOMAIN = `${API_URL}/ro-rule/get-by-domain`
export const GET_RO_RULE_BY_ID = `${API_URL}/ro-rule/get-by-id`
export const GET_DELETE_RULES_BY_ID = `${API_URL}/ro-rule/delete`
</pre>

Tất cả các endpoint sử dụng trong project sẽ lưu tại đây và được export ra và call http-request tại từng nơi.

<i>Thay đổi này sẽ thực hiện ở tất các các http-request hiện tại</i>

**2.Sử dụng 1 helper cho việc call các http-request** 

example http.js

<pre>

/**
 * helper http-request
 * @param urlPath
 * @param bodyData
 * @param options
 * @returns {Promise<any>}
 */
export const request = async (urlPath, bodyData, options = {}) => {
    let response = await fetch(urlPath, {
        method: options?.method ? options?.method: 'POST',
        headers: options?.headers ? options?.headers : { 'Content-Type': 'application/json' },
        body: JSON.stringify(bodyData)
    })
    return await response.json()
}
</pre>

VD khi call 1 request: 

<pre>
import * as httpFacade from "./../../helper/http"

/**
 * @param domain
 * @param token
 * @param apiVersion
 * @param dispatch
 */
export const getShopConfig = (domain, token, apiVersion, dispatch) => {
    httpFacade.request(httpFacade.GET_CONFIGURATION, {domain, token, apiVersion })
        .then(r => {
            if (r.success) {
                ====== Logic here ========

                dispatch(getConfigSuccess(payload))

            } else {
                reduxGetDataLog(domain, 'FAILED', domain, 'Failed get CONFIGURATIONS Location: redux/act/configurations.js line 8')
            }
        })
        .catch(err => {
            reduxGetDataLog(domain, 'FAILED', err, 'Failed get CONFIGURATIONS Location: redux/act/configurations.js line 4')
        })
}
</pre>

**3.Tách các http-request thành từng service riêng**

<img src="/folder.png" alt="example folder">

Mỗi service làm đúng vai trò là thực hiện các http-request, đặc biệt với các func dùng ở nhiều nơi (lấy danh sách product, danh sách customer, product tags, customer tags,...) thì có thể xử lý phần logic ngay trong func service

example:

<pre>
import * as httpFacade from "../helper/http";
import {reduxGetDataLog} from "../log";

const apiVersion = API_VERSION

/**
 *
 * @param domain
 * @param token
 * @param isConfigurationPage
 * @returns {Promise<boolean>}
 */
export const getConfigResource = async ({domain, token, isConfigurationPage}) => {
    let configuration = false
    await httpFacade.request(httpFacade.GET_CONFIGURATION, {domain, token, apiVersion})
        .then(r => {
            configuration = isConfigurationPage ? r : false

        })
        .catch(err => {
            reduxGetDataLog(domain, 'FAILED', err, 'Failed get RESOURCE_CONFIGURATION Location: services/configService.js line 13')
        })

    return configuration
}
</pre>

Lúc đấy tại dispatcher (saga or toolkit) việc thực hiện call request sẽ gắn gọn hơn

example với action call resource data tại saga (func nhiều dòng nhất)

<pre>
/**
 * 
 * @param domain
 * @param token
 * @param apiVersion
 * @param dispatch
 * @param id
 * @returns {Promise<void>}
 */
export const getResourceData = async ({domain, token, apiVersion, dispatch, id}) => {
    let realId = id !== "add";
    let isConfigurationPage = id == null;

    /**
     * Fetch shop configuration
     */
    let configuration = await getConfigResource({domain, token, isConfigurationPage})

    /**
     * Fetch rule
     */
    let rule = await getRuleByIdResource({domain, id, realId})

    /**
     * Fetch Customer Tag
     */
    let customerTags = await getCustomerTagsResource({domain, token})

    /**
     * Fetch Product Tags
     */
    let productTags = await getProductTagResource({domain, token})

    /**
     * Fet Products
     */
    let products = await getProductResource({domain, token, rule})

    /**
     * Fetch customers
     */
    let customers = await getCustomerResource({domain, token, rule, configuration})

    /**
     * Fetch product collections
     */
    let collections = await getProductCollectionResource({domain, token})

    /**
     * Dispatch
     */
    dispatch(getQoResourceDataSuccess({
        domain,
        collections,
        products,
        productTags,
        customers,
        customerTags
    }))
}
</pre>


