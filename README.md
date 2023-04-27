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

<img src="data:image/png;base64,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" alt="example folder">
