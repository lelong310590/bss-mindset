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
    yield takeLatest(QO_SAVE_RULE_ASYNC, saveQoRule);
    yield takeLatest(GET_RULE_BY_ID_ASYNC, getRule);
}
</pre>

**3 Các bước xử lý call dispatch và action phía page và components ...**
