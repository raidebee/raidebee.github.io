---
title: O2O 서비스의 결제 마이크로서비스 개발기
description: 
author: ray
date: 2020-12-01 00:00:00 +0900
categories: [개발기]
tags: [결제 서비스]
pin: false
mermaid: true
---

미소(O2O 홈서비스 스타트업)에 입사한 직후, 기존 모놀리식 서비스의 결제 모듈을 마이크로서비스로 전환하는 작업을 맡게 되었습니다. 기존 결제 모듈은 - 아마도 빠른 출시를 위해서 - 서비스를 위한 최소한의 기능만 구현된 상태로 수년간 사용되고 있었고, 구현되지 않은 기능은 단순 작업을 반복하거나 편법으로 처리하고 있었습니다. 따라서 기능을 업데이트하는 것은 물론, 전반적인 완성도를 높이는 작업이 필요했습니다.

### 마이크로서비스 인증

![요청 흐름도](/assets/posts/2020-12-01-payment-service/request-flow-by-user-type.png){: width="700" }
_요청 흐름도_

사내에 마이크로서비스 인증을 위한 모듈인 'auth proxy'가 이미 개발되어 있었습니다. 이는 OpenResty 플랫폼을 마이크로서비스에서 사용할 수 있도록 수정한 것이었습니다. 결제 서비스의 사용자는 고객, 파트너, 백오피스로 나뉘는데, API 요청 시 헤더에 사용자의 Bearer 토큰을 포함해야 합니다. 이 토큰이 인증용 Redis의 Key로 존재하면, Value에 저장된 사용자 ID를 다시 헤더에 설정해서 요청합니다. 그렇지 않으면 모놀리식 서비스의 인증 API를 사용해 해당 토큰을 검증한 후 같은 처리를 했습니다. 토큰이 유효하지 않으면 401, 해당 리소스에 대한 접근 권한이 없으면 403 상태 코드를 반환하게 됩니다. 이 인증 처리 로직은 Lua 스크립트로 작성되어 있었습니다.

사용자 토큰은 주기적으로 갱신되기 때문에 일반 사용자가 아닌 서버 간 요청 시에는 사용하기가 어려웠습니다. 그래서 서버 간 인증에는 Django REST Framework의 API Key 모듈을 사용했습니다. 기본적으로 같은 VPC 내에서 private load balancer를 통해 요청하며, 이때 헤더에 API Key를 포함해야 합니다. 이 요청은 2080 포트를 통해 auth proxy로 전달되고, Lua 스크립트를 이용한 인증 처리를 우회한 뒤 Django에서 API Key로 인증하게 됩니다.

Django에서는 DRF의 `authentication.BaseAuthentication` 클래스를 상속해서 `authenticate` 함수를 오버라이딩했고, 여기서 auth proxy에 의해 헤더에 설정된 사용자 정보를 토대로 `User` 인스턴스를 반환합니다. 이 클래스를 각 view의 `authentication_classes`에 추가해서 인증하고, 각 view에 대한 사용자별 권한은 `permission_classes`를 통해 처리합니다. 예를 들어, 모든 사용자가 접근할 수 있는 view의 `permission_classes`는 다음과 같이 지정합니다.

```python
permission_classes = (
    CustomerPermission | SupplierPermission | BackofficePermission | HasAPIKey,
)
```
{: .nolineno }

### 결제 수단 구조 개선

기존 결제 모듈은 계좌이체, 예치금, 나이스페이 등을 결제 수단으로 지원했고, 다중 PG사 지원을 고려한 구조는 아니었습니다. 계좌이체와 예치금은 결제 수단이 맞지만, 나이스페이는 PG사 이름이고 실제 결제 수단은 빌링이어야 했습니다. 서로 다른 레벨의 이름이 같은 레벨에 존재하고 있어, 이 문제를 해결하기 위해 결제 수단 속성에 '서비스 제공자'를 추가했습니다. 계좌이체와 현금은 수동으로 처리하므로 서비스 제공자는 '수동(사람)'이고, 예치금은 새로 개발된 미소의 예치금 서비스를 사용하므로 서비스 제공자는 '미소'입니다.

💳 **기존 결제 모듈의 결제 수단**

계좌이체, 현금, 나이스페이,
예치금

```javascript
const PAYMENT_METHODS = {
  cash: '현금',
  bank: '계좌이체',
  deposit: '예치금',
  nicepay: 'nicepay',
}
```
{: .nolineno }

💳 **결제 마이크로서비스의 결제 수단**

나이스페이 일반: 미등록 카드, 가상계좌<br/>
나이스페이 빌링: 등록 카드<br/>
수동: 계좌이체, 현금<br/>
미소: 예치금

```javascript
const PAYMENT_METHODS = [
  {
    name: '나이스페이 일반-카드',
    serviceId: SERVICE.NICEPAY_GENERAL,
    method: PAYMENT_METHOD.CARD,
  },
  {
    name: '나이스페이 일반-가상계좌',
    serviceId: SERVICE.NICEPAY_GENERAL,
    method: PAYMENT_METHOD.VBANK,
  },
  {
    name: '나이스페이 빌링-등록카드',
    serviceId: SERVICE.NICEPAY_RECURRENT,
    method: PAYMENT_METHOD.BILL,
  },
  {
    name: '수동-계좌이체',
    serviceId: SERVICE.PERSON_MANUAL,
    method: PAYMENT_METHOD.BANK,
  },  
	{
    name: '수동-현금',
    serviceId: SERVICE.PERSON_MANUAL,
    method: PAYMENT_METHOD.CASH,    
  },
  {
    name: '미소-예치금',
    serviceId: SERVICE.MISO_CREDIT,
    method: PAYMENT_METHOD.CREDIT,
  },
];
```
{: .nolineno }

### 결제 서비스 처리 추상화

![결제 서비스 처리 클래스 다이어그램](/assets/posts/2020-12-01-payment-service/payment-provider-class-diagram.png)
_결제 서비스 처리 클래스 다이어그램(초기 버전이어서 현재와 차이가 있으며, 스트래티지 패턴은 구현되지 않았음)_

기존 결제 모듈에서는 결제 수단에 따라 하나의 모듈 내에서 분기하는 방식으로 처리하고 있었습니다. 구조가 간단해서 유지보수가 어렵지는 않았지만, 앞으로 여러 결제 서비스가 추가될 것을 고려하면 결제 서비스별 처리를 추상화하는 작업이 반드시 필요했습니다.

보통 PG사들이 일반 카드 결제와 빌링 결제를 다른 서비스로 분류하기 때문에, 일반 결제와 빌링 결제 서비스 인터페이스를 각각 만들었습니다. 현재는 나이스페이의 빌링 결제만 지원하지만, 나중에 이니시스의 빌링 결제를 지원하려면 나이스페이와 같이 빌링 결제 서비스 인터페이스를 상속해서 구현하면 됩니다. 기본 서비스 인터페이스의 주요 함수는 `pay`(결제)와 `cancel`(취소)이고, 이 인터페이스를 상속한 빌링 서비스 인터페이스에는 `create_key`(빌링키 생성)와 `destroy_key`(빌링키 삭제) 함수가 추가되었습니다.

```python
class BasePaymentService(ABC):
    @property
    @abstractmethod
    def config(self):
        pass

    @abstractmethod
    def pay(self, *args, **kwargs):
        pass

    @abstractmethod
    def cancel(self, *args, **kwargs):
        pass

    @abstractmethod
    def get_status(self, *args, **kwargs):
        pass
```
{: .nolineno }

```python
class BaseRecurrentPaymentService(BasePaymentService):
    @abstractmethod
    def create_key(self, credit_card: CreditCard):
        pass

    @abstractmethod
    def destroy_key(self, key: BillingKey):
        pass
```
{: .nolineno }

### DRF View에서 비즈니스 로직 분리하기

대부분의 View는 DRF의 `ModelViewSet`을 사용해서 구현했습니다. 단순히 CRUD를 지원하는 REST API를 제작하고 싶다면 Django 모델로 `ModelSerializer`를 생성하고, 이것을 `ModelViewSet`에 지정하기만 하면 됩니다. 그런데 비즈니스 로직이 추가되어야 한다면 어떻게 해야 할까요? 안티패턴은 View 내에 로직을 추가하는 것입니다. 비즈니스 로직이 비교적 간단하다면 모델 내에 두는 것이 베스트 프랙티스일 것입니다. 하지만 결제 처리는 그렇게 간단하지 않기 때문에, 서비스 레이어를 만드는 것이 가장 괜찮아 보이는 방법이었습니다. 기본적인 CRUD는 DRF에 맡기고, 비즈니스 로직은 서비스 레이어에서 처리하도록 만들어야 했습니다. (스택오버플로우와 여러 블로그를 참고했는데, 최종적으로는 C\#을 사용한 MVC 아키텍처 글의 방법을 거의 비슷하게 따랐습니다.) 결론적으로 모델 내에서 서비스를 호출하고, 서비스에서 비즈니스 로직을 구현했습니다. 참고로 'service'라는 이름을 결제 서비스(예: PG사 서비스)용으로 쓰고 있어서, 서비스 레이어의 이름은 조금 투박해 보이지만 'process'로 짓게 되었습니다.

다음은 결제 처리 시 공통으로 사용하는 서비스 클래스의 일부분입니다.

```python
class CommonPaymentProcess(
    PaymentProcess, LoggingMixin, OrderNameHelper, RequestUserHelper
):
    def __init__(self, instance, service):
        self.instance = instance
        self.service = service
        self.set_logger(name=self.__class__.__name__)

    def create(self):
        if self.instance.payment_type == PAYMENT_TYPE.PAY:
            return self._create_pay()
        if self.instance.payment_type == PAYMENT_TYPE.CANCEL:
            return self._create_cancel()

    def _create_pay(self):
        self._create_common_payment()

    def _create_cancel(self):
        self._create_common_payment()

    def update(self):
        raise UnsupportedFunctionError

    def submit(self):
        if self.instance.payment_type == PAYMENT_TYPE.PAY:
            return self._submit_pay()
        if self.instance.payment_type == PAYMENT_TYPE.CANCEL:
            return self._submit_cancel()

    def _submit_pay(self):  # pragma: no cover
        raise NotImplementedError

    def _submit_cancel(self):  # pragma: no cover
        raise NotImplementedError
```
{: .nolineno }

빌링 결제는 위 클래스를 상속해서 아래와 같이 구현했습니다. 이 클래스는 나이스페이 빌링뿐만 아니라, 다른 PG사의 빌링 서비스를 연동할 때도 공통으로 사용할 수 있습니다. 위에서 설명한 결제 서비스 처리의 추상화를 통해 이것이 가능해졌습니다. `self.service`가 각 PG사의 서비스를 연동한 클래스의 인스턴스이고, `pay` 함수를 통해 결제를 요청합니다.

```python
class RecurrentPaymentProcess(CommonPaymentProcess):
    def _submit_pay(self):
        customer = Customer(
            self.get_user_name(self.instance.user_type, self.instance.user_id),
            self.instance.key.key,
        )
        order = CardOrder(
            self.instance.order_uid_with_user_uid,
            self.get_order_name(self.instance.order_type, self.instance.order_id),
            self.instance.amount,
        )
        self.result = self.service.pay(customer, order)
        self.update_instance_from_result()
        return (
            PAYMENT_STATE.REJECTED
            if self.result.is_rejected
            else PAYMENT_STATE.CONFIRMED
        )

    def _submit_cancel(self):
        order = CancellationOrder(
            self.instance.origin.order_uid_with_user_uid,
            self.instance.origin.tx_id,
            self.instance.cancel_reason,
            self.instance.amount,
            is_partial_cancel=self.instance.is_partial_cancel,
        )
        self.result = self.service.cancel(order)
        self.update_instance_from_result()
        return (
            PAYMENT_STATE.REJECTED
            if self.result.is_rejected
            else PAYMENT_STATE.CONFIRMED
        )
```
{: .nolineno }

아래는 나이스페이의 빌링 서비스를 연동한 클래스의 일부입니다.

```python
class NicepayRecurrentPaymentService(LoggingMixin, BaseRecurrentPaymentService):
    def __init__(self, mid=None, key=None, test_mode=settings.TEST_MODE, verbose=False):
        self._config = NicepayRecurrentPaymentConfig(
            mid=mid, key=key, test_mode=test_mode, verbose=verbose
        )
        self.set_logger(name=self.__class__.__name__)

    @property
    def config(self):
        return self._config

    def pay(self, customer: Customer, order: Order):
        model = NicepayRecurrentPaymentPayModel(customer, order)
        payload = vars(model)
        payload.update(
            {
                "MID": self.config.MID,
                "EncodeKey": self.config.KEY,
                "actionType": "PYO",
                "EncryptedData": self.config.KEY,
                "TID": self.generate_tid(),
                "CancelPwd": self.config.CANCEL_PASSWORD,
            }
        )
        response = self.request(
            REQUEST_VERB.POST,
            self.config.build_uri("pay"),
            data=payload,
            **self.config.DEFAULT_REQUEST_KWARGS,
        )

        return NicepayRecurrentPaymentServicePayResult(
            response, params={"customer": vars(customer), "order": vars(order)}
        )
```
{: .nolineno }

다음과 같이 View의 `create` 함수에서 `_create_by_process` 함수를 통해 모델의 `create` 함수를 호출하게 됩니다.

```python
    def create(self, request, *args, **kwargs):
        request_data = request.data.copy()
        self._create_action(request, request_data)

        self.update_request_data_from_request(request, request_data)
        serializer = self.get_serializer(data=request_data)
        serializer.is_valid(raise_exception=True)

        instance = self.perform_new(serializer)
        instance = self._create_by_process(instance)
        serializer = self.serializer_class(instance)
        headers = self.get_success_headers(serializer.data)
        response = serializer.data
        return Response(response, status=status.HTTP_201_CREATED, headers=headers)

    @handle_service_exceptions
    def _create_by_process(self, instance):
        instance.create()
        return instance
```
{: .nolineno }

모델의 `create` 함수는 다음과 같이 인스턴스의 `service`(예: 나이스페이 일반, 빌링)에 따라 위에서 구현한 서비스 클래스를 호출하게 됩니다. 아래 코드에서 `process_class`가 서비스 클래스이고, `service.service_instance`는 개별 서비스(예: 나이스페이 일반, 빌링)를 연동한 클래스의 인스턴스입니다. 예를 들어 나이스페이 빌링 결제를 처리할 때 `process_class`는 `RecurrentPaymentProcess`이고, `service.service_instance`는 `BaseRecurrentPaymentService`를 상속해서 구현한 `NicepayRecurrentPaymentService`의 인스턴스입니다.

```python
    def create(self):
        process = self.process_class(self, self.service.service_instance)
        process.create()
```
{: .nolineno }

**정리하면 View의 create 처리는 다음과 같습니다.**

View의 `create` 함수에서 모델의 `create` 함수를 호출하고, 이 함수는 비즈니스 로직을 처리하는 서비스 클래스의 `create` 함수를 호출합니다. 이때 해당 인스턴스와 개별 서비스 처리 인스턴스를 인자로 전달합니다.

### 다중 결제 수단 지원

기존 결제 모듈에서는 한 명의 사용자가 동시에 하나의 결제 수단만 가질 수 있었습니다. 결제 수단을 변경하면 이전 결제 수단 정보는 삭제되었습니다. 예를 들어, 빌링 결제 카드를 등록하고 나서 예치금으로 결제 수단을 변경하면 기존 카드 정보가 삭제되었습니다.

새로운 결제 서비스에서는 동시에 여러 개의 결제 수단을 가질 수 있고, 결제 시 등록된 결제 수단 중에서 선택해서 결제할 수 있습니다.

![주문 결제](/assets/posts/2020-12-01-payment-service/modal-pay.png){: width="500" }
_주문 결제_

### 다중 카드 지원

기존 결제 모듈은 한 명의 사용자가 단 하나의 카드만 등록할 수 있는 한계를 가지고 있었습니다. 새로운 카드를 추가하면 이전 카드 정보는 삭제되는 구조였습니다.

새롭게 개발한 결제 서비스에서는 이 점을 개선하여, 한 명의 사용자가 여러 장의 카드를 등록하고 관리할 수 있도록 했습니다. 이 중 하나의 카드를 '기본 카드'로 지정하면 후불 결제(자동 결제)에 사용됩니다. 고객이 직접 결제하는 수동 결제의 경우에는, 등록된 카드 외에도 스마트폰 앱카드나 가상계좌 등 다양한 결제 수단을 자유롭게 이용할 수 있도록 구현했습니다.

### 자동/수동 결제 수단 설정 분리

기존 결제 모듈에서는 자동 결제 여부가 사용자가 선택한 결제 수단에 따라 결정되었습니다. 예를 들어, 나이스페이나 예치금처럼 자동 결제를 지원하는 수단을 등록하면 서비스 완료 후 자동으로 결제가 이루어졌습니다. 반면, 자동 결제를 원하지 않을 경우 사용자는 카드 정보를 삭제하고 계좌이체와 같은 다른 수단으로 변경해야 하는 불편함이 있었습니다.

결제 서비스에서는 자동 결제와 수동 결제 설정을 명확히 분리했습니다. 사용자는 자동 결제를 지원하는 결제 수단 중에서 원하는 것을 선택하고, 자동 결제 기능 자체를 켜거나 끌 수 있습니다. 이를 통해 사용자의 기존 결제 정보를 유지하면서도, 자동 결제 여부를 훨씬 더 유연하게 관리할 수 있게 되었습니다.

![자동 결제 설정 변경](/assets/posts/2020-12-01-payment-service/modal-auto-payment-setting.png){: width="400" }
_자동 결제 설정 변경_

```json
{
  "id": 233618,
  "user_type": "customer",
  "user_id": 672815,
  "auto_payment_service_id": 2,
  "auto_payment_method": "bill",
  "manual_payment_service_id": null,
  "manual_payment_method": "",
  "is_auto_payment_activated": true
}
```
{: .nolineno }

### 결제 거래 모델

기존 결제 모듈은 결제 데이터를 관리하는 데 몇 가지 아쉬운 점이 있었습니다. 결제 수단에 따라 저장되는 정보가 제각각이었고, 결제를 취소하면 해당 데이터를 삭제하는 방식으로 처리했습니다. 이 때문에 동일한 주문에 대해 결제와 취소를 반복하면 최종 기록만 남았고, 만약 최종 상태가 '취소'라면 아무런 기록도 남지 않는 문제가 있었습니다.

결제 서비스에서는 이를 개선하기 위해 결제 거래 모델을 새롭게 설계했습니다. 먼저, 결제 수단별로 저장 항목을 표준화했습니다. 그리고 ‘주문 결제 모델’과 ‘결제 거래 모델’을 분리하여, 결제나 취소가 발생할 때마다 모든 내역을 ‘결제 거래’ 테이블에 빠짐없이 기록하도록 했습니다. 덕분에 여러 번의 부분 취소나 결제 실패가 발생하더라도 모든 과정을 추적할 수 있는 안정적인 구조를 갖추게 되었습니다.

  * 동일 주문에 대한 결제와 취소 반복 기록 (백오피스 화면)

    ![백오피스 주문 정산 상세 화면](/assets/posts/2020-12-01-payment-service/all-txs.png){: width="350" }

  * 부분 취소 거래 데이터 예시  
    (52,600원 결제 → 1차 10,000원 부분 취소 → 2차 20,000원 부분 취소)

    ```json
    {
      "tx": {
        "id": 1544094,
        "cancellation_txs": [
          {
            "id": 1544096,
            "payment_type": "cancel",
            "amount": "20000.00",
            "state": "confirmed",
            "confirmed_at": "2020-10-12T01:52:07.579302+09:00",
            "created_at": "2020-10-12T01:52:07.534838+09:00",
            "updated_at": "2020-10-12T01:52:07.579519+09:00"
          },
          {
            "id": 1544095,
            "payment_type": "cancel",
            "amount": "10000.00",
            "state": "confirmed",
            "confirmed_at": "2020-10-12T01:51:57.624504+09:00",
            "created_at": "2020-10-12T01:51:57.590709+09:00",
            "updated_at": "2020-10-12T01:51:57.624698+09:00"
          }
        ],
        "service_id": 5,
        "payment_type": "pay",
        "payment_method": "credit",
        "user_type": "customer",
        "user_id": 847898,
        "order_type": "booking",
        "order_id": 2105558,
        "amount": "52600.00",
        "remaining_amount": "22600.00",
        "state": "refunded_partially",
        "confirmed_at": "2020-10-12T01:51:42.465832+09:00",
        "created_at": "2020-10-12T01:51:42.410108+09:00",
        "updated_at": "2020-10-12T01:52:07.595322+09:00"
      }
    }
    ```
    {: .nolineno }

여러 주문을 묶어서 한 번에 결제하는 경우에는 결제 거래 테이블 `payment_transaction`에 전체 금액을 결제한 레코드가 생성되고 주문별 결제 레코드 payment에서 이 레코드를 참조합니다. 주문 결제 레코드에서 최종 결제 정보를 가져오는 것으로 대부분의 처리를 할 수 있고 주문별 상세 결제 내역이 필요한 일부 기능에서만 결제 거래 정보를 가져오면 됩니다.

![주요 모델 관계도](/assets/posts/2020-12-01-payment-service/payment-service-models.png){: width="700" }
_주요 모델 관계도_

### 결제 상태 관리

![결제 거래 모델의 상태 다이어그램](/assets/posts/2020-12-01-payment-service/payment-transaction-states-diagram.png)
_결제 거래 모델의 상태 다이어그램_

결제 거래의 상태는 **Finite-State Machine (FSM)** 패턴을 적용하여 관리했습니다. 예를 들어, `prepared` 상태의 결제가 `confirmed` 상태로 바로 전환될 수 없고, 반드시 `before_submitting` 단계를 거치도록 강제하는 방식입니다. 이를 통해 임의의 상태 변경을 막아 데이터의 무결성을 보장할 수 있었습니다.

![결제 거래 모델 상태의 트랜지션들](/assets/posts/2020-12-01-payment-service/payment-transaction-transitions.png){: width="400" }
_결제 거래 모델 상태의 트랜지션들_

구현에는 `django-fsm` 모듈을 활용했습니다. 과거 가상화폐 거래소 개발 당시 사용했던 Rails의 `aasm` 패키지는 상태 전환 전후에 콜백(callback)을 직접 설정할 수 있어 직관적이었는데, `django-fsm`은 동일한 기능을 구현하려면 시그널(signal)을 사용해야 하는 약간의 차이가 있었습니다.

* 결제 모델 내 django-fsm 사용 부분
  ```python
  def can_submit(self):
      if (
          self.service.service.uid == PAYMENT_SERVICE.GENERAL_PAYMENT
          and self.payment_type == PAYMENT_TYPE.PAY
      ):
          return self.state == self.STATE.RECEIVED_TOKEN
      return self.state == self.STATE.NEW

  @transition(
      field=state,
      source=[STATE.NEW, STATE.RECEIVED_TOKEN],
      target=STATE.BEFORE_SUBMITTING,
      conditions=[can_submit],
  )
  def before_submitting(self):
      logger.log.debug(f"BEFORE_SUBMITTING transition of {self} was fired.")

  @transition(
      field=state,
      source=STATE.BEFORE_SUBMITTING,
      target=RETURN_VALUE(STATE.PRE_CONFIRMED, STATE.CONFIRMED, STATE.REJECTED),
      on_error=STATE.FAILED,
  )
  def submit(self):
      logger.log.debug(f"SUBMIT transition of {self} was fired.")
      process = self.process_class(self, self.service.service_instance)
      return process.submit()

  @transition(
      field=state,
      source=[STATE.BEFORE_SUBMITTING, STATE.PRE_CONFIRMED],
      target=STATE.CONFIRMED,
  )
  def confirm(self):
      logger.log.debug(f"CONFIRM transition of {self} was fired.")
  ```
  {: .nolineno }

결제 서비스에서 결제 거래 모델의 주요 상태 명세는 다음과 같습니다.

**결제 거래 모델 상태**

| 상태                        | 설명                                                                                                                                      |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| prepared                    | 고객이 직접 결제하는 일반 결제에서 고객이 결제 버튼을 클릭한 상태입니다.                                                                  |
| before_submitting           | PG사로 결제 요청을 보내기 직전 업데이트 되는 상태입니다.<br>이 상태에서 멈춰 있다면 PG사로 결제 요청을 보낸 후 응답 받지 못한 상태입니다. |
| pre_confirmed               | 가상계좌 발급 후 미입금 상태입니다.                                                                                                       |
| confirmed                   | 결제 완료된 상태입니다.                                                                                                                   |
| refunded/refunded_partially | 전체 취소/부분 취소 완료된 상태입니다.                                                                                                    |
| rejected                    | 카드 유효 기간 오류, 잔액 부족 등의 사유로 결제 거부된 상태입니다.                                                                        |

> **"결제했는데 왜 완료가 안 되나요?"**
>
> 가끔 고객이 결제를 완료했다고 생각하지만, 시스템에는 `prepared` 상태로 남아있는 경우가 있었습니다. 이는 고객이 결제 버튼을 클릭한 후, 최종 완료 단계까지 진행하지 않았을 때 발생합니다. 이런 문의를 받을 때마다 제가 **"고객님, 결제 버튼은 클릭하셨지만 아직 최종 완료 처리는 되지 않은 상태입니다."** 라고 안내해 드린 경험이 여러 번 있습니다.

### 결제 통신 오류 시 처리

기존 결제 모듈에서는 PG사 결제는 성공했지만, 네트워크 오류 등으로 데이터베이스에 저장이 실패하면 이중 결제가 발생하는 문제가 있었습니다. 고객의 문의가 접수되어야 상황을 파악하고 수동으로 결제를 취소해야 했습니다.

새로운 결제 서비스에서는 이런 문제를 방지하기 위해, 결제 상태가 `before_submitting`에 머물러 있으면 2분 이내에 PG사의 결제 통보(Webhook)를 통해 최종 성공 여부를 확인하고 상태를 업데이트하도록 했습니다. 또한, 결제 통보가 한 번이라도 실패하면 Sentry를 통해 즉시 알림이 오기 때문에, 오류 상황을 놓치지 않고 대응할 수 있습니다. 덕분에 결제가 완료되었으나 데이터가 누락되어 이중 결제되는 상황이 발생하지 않게 되었습니다.

### 결제 취소 지원

기존 모듈은 결제 취소 기능을 지원하지 않아, 나이스페이 결제 건을 취소하려면 관리자 페이지에 직접 접속해서 처리하고, 백오피스에서 수동으로 결제 정보를 삭제해야 하는 번거로움이 있었습니다.

새로운 결제 서비스에서는 백오피스에서 직접 **전체 취소**와 **부분 취소**를 모두 지원하도록 구현했습니다.

![주문 결제 취소](/assets/posts/2020-12-01-payment-service/modal-cancel.png){: width="500" }
_주문 결제 취소_

### 예치금 서비스

기존 예치금 기능은 단순히 수동으로 금액을 설정하고 차감하는 수준에 머물러 있었습니다. 거래 내역이 별도로 기록되지 않았고, 환불 시 잔고가 자동으로 업데이트되지 않는 등 기능적 제약이 많았습니다.

결제 서비스에서는 이를 **네이버페이나 쿠팡 수준의 예치금 기능**으로 대폭 개선했습니다. 일회성 충전은 물론, 잔액이 일정 금액 이하로 내려가면 자동으로 충전되는 '기준 하한 충전'과 매월 정해진 금액을 충전하는 '월 정액 충전' 기능을 지원합니다. 또한, 모든 충전 및 결제 내역을 기록하고 조회할 수 있으며, 잔고 내에서 간편하게 환불할 수도 있습니다.

![예치금 자동 충전 설정 - 기준 하한 자동 충전](/assets/posts/2020-12-01-payment-service/modal-credit-lower-limit.png){: width="500" }
_예치금 자동 충전 설정 - 기준 하한 자동 충전_

![예치금 자동 충전 설정 - 월 정액 자동 충전](/assets/posts/2020-12-01-payment-service/modal-credit-monthly-fixed.png){: width="500" }
_예치금 자동 충전 설정 - 월 정액 자동 충전_

![예치금 관리 - 거래 내역](/assets/posts/2020-12-01-payment-service/credit-txs.png){: width="700" }
_예치금 관리 - 거래 내역_

결제 거래의 경우 기본적으로 insert만 가능하고 update를 하는 경우도 있지만 동시에 하지는 않기 때문에 race condition이 발생하지 않습니다. 만약 발생할 가능성이 있다면 간단히 `django-fsm` 모듈의 optimistic locking mixin을 상속하면 예방할 수 있습니다. 

예치금 잔액을 업데이트할 때는 여러 결제가 동시에 처리될 때 발생할 수 있는 race condition을 방지하기 위해, Django의 `select_for_update`를 사용하여 데이터의 정합성을 확보했습니다.

```python
def _update_account_balance(self):
    account = (
        Account.objects.using("default")
        .select_for_update()
        .get(id=self.instance.account.id)
    )

    if self.instance.payment_type == PAYMENT_TYPE.PAY:
        account.sub_funds(
            self.instance.amount,
            reason=ACCOUNT_VERSION_REASON.PAYMENT_COMPLETED,
            ref=self.instance,
        )
    else:
        account.add_funds(
            self.instance.amount,
            reason=ACCOUNT_VERSION_REASON.PAYMENT_REFUNDED_PARTIALLY
            if self.instance.is_partial_cancel
            else ACCOUNT_VERSION_REASON.PAYMENT_REFUNDED,
            ref=self.instance,
        )
```
{: .nolineno }

### 결제 링크 기능

기존에는 미납 금액이 발생한 고객에게 회사의 계좌 번호를 안내하고 직접 이체하도록 하는 방식을 사용했습니다. 이 방식은 입금자명을 확인하고 수동으로 처리해야 했기 때문에, 고객과 파이낸스팀 모두에게 불편함이 있었습니다. 입금자가 다르거나 동명이인이면 처리가 더 지연됐습니다.

![결제 링크 기능](/assets/posts/2020-12-01-payment-service/bill-link-screen.png)
_결제 링크 기능_

이 문제를 해결하기 위해, 주문별 **결제 전용 웹페이지 링크**를 문자로 전송하는 방식을 도입했습니다. 고객은 이 페이지에서 스마트폰 앱카드나 가상계좌로 즉시 결제할 수 있으며, 결제 상태(미결제, 입금 대기, 결제 완료)를 실시간으로 확인할 수 있습니다. 덕분에 더 이상 고객이 결제 처리 여부를 확인하기 위해 고객센터에 연락할 필요가 없어졌습니다.

- 결제 상태에 따라 결제 페이지 접속 시 보여주는 내용
  - 미결제 시: 이용 서비스 상세 내역, 결제 버튼
  - 가상계좌 발급 후 미입금 시: 가상계좌 입금 정보
  - 결제 완료 시: 결제 완료



![결제 링크 - 미결제 시](/assets/posts/2020-12-01-payment-service/bill-link-1.png){: width="300" }
_결제 링크 - 미결제 시_

![결제 링크 - 가상계좌 발급 후 미입금 시](/assets/posts/2020-12-01-payment-service/bill-link-2.png){: width="300" }
_결제 링크 - 가상계좌 발급 후 미입금 시_

> 참고로 위 결제 링크 화면은 디자이너인 팀 디렉터가 작업했습니다.
> 
> 처음에 이 화면을 작업할 디자이너가 없었기 때문에 팀의 시니어 개발자가 일단 내가 직접 간단하게 작업해보라고 해서 아래와 같이 디자인했습니다. (스케치를 처음 사용해봤고 하루 정도 걸렸습니다.)
> 
> ![미납 요금 안내](/assets/posts/2020-12-01-payment-service/receipt.png){: width="300" }
  _미납 요금 안내_
>
> 사내 프론트엔드 개발자가 이 디자인을 보고 현재 서비스 페이지와 톤앤매너가 맞지 않다는 의견을 줘서 다음과 같은 메시지를 디자이너인 팀 디렉터에게 보냈습니다.
>
> > 미납 요금 안내 화면 제플린 초대해드렸습니다. 현재 미소 디자인과 많이 다르다는 의견이 있어서 어떻게 진행하면 좋을지 의견 주세요. 스케치 파일을 공유해 주시면 제가 기존 리소스들을 가지고 조립하는 방식으로 작업하는 것도 가능할 것 같습니다.
>
> 이 후 시니어 개발자, 디자이너와 논의 끝에 기존 컨셉에 맞고 중요한 정보만 보여주는 방식으로 변경하게 되었습니다. 입사한 지 한 달도 안된 시점이었고 고객 앱 외의 화면들은 꾸밈이 거의 없는 심플한 페이지였기 때문에 컨셉을 제대로 이해하지 못했던 것 같습니다.

### 기존 모놀리식 서비스 → 결제 서비스 마이그레이션

![마이그레이션 회차별 레코드](/assets/posts/2020-12-01-payment-service/misoweb-migration.png)
_마이그레이션 회차별 레코드_

결제 서비스를 처음 배포할 당시, 기존 모놀리식 서비스에 쌓인 결제 정보는 약 200만 건에 달했고, 이 데이터를 새로운 서비스로 이전하는 마이그레이션을 진행했습니다. 또한, 배포 이후에도 기존 시스템에서 발생하는 결제 정보를 지속적으로 새로운 서비스로 이전해야 했습니다.

이를 위해 `Celery beat`를 사용하여 3분 간격으로 마이그레이션 작업을 실행했으며, 처리 방식은 다음과 같습니다.

  * **마이그레이션 대상 선정**: 특정 시간 범위 내에 업데이트된 레코드 ID 목록을 가져와 정해진 배치 사이즈(예: 10,000개)로 나누어 처리합니다.
    1. 저장 시 지연되는 경우를 고려해 업데이트 시각이 이전 마이그레이션 종료 시점부터 현재 시각 n초 전(기본 15초) 사이인 레코드의 전체 ID 목록을 불러옵니다.
        ```json
        [1, 2, 3, 4, 5, 6, 7, 8, 9, 10,...]
        ```
        {: .nolineno }
    2. 전체 ID 목록을 설정된 배치 사이즈(기본 10,000개)로 나눕니다.
        ```json
        [[1, 2, 3, 4, 5,...], [10001, 10002, 10003, 10004, 10005,...]]
        ```   
        {: .nolineno }
    3. 배치별 처리 방법
         * 배치별 ID 목록으로 - where in을 사용해 - 레코드들을 불러옵니다.
         * 만약 레코드의 업데이트 시각이 전체 ID를 불러온 시점(1) 이후이면 처리하지 않습니다. 다음 회차 마이그레이션에서 처리될 것입니다.
  * **중복 실행 방지**: 이전 마이그레이션 작업이 아직 진행 중일 경우, 새로운 작업을 시작하지 않도록 하여 중복 실행을 막았습니다. 진행이 지연될 경우 Sentry를 통해 즉시 보고받았습니다. 주로 모놀리식 서비스의 GraphQL 서버에서 timeout 오류가 발생한 경우였습니다.
    
    ![중복 실행 방지](/assets/posts/2020-12-01-payment-service/migration-in-progress-sentry.png){: width="400" }
    _마이그레이션 시작 시 이전 마이그레이션이 진행 중일 때 Sentry로 보고합니다._

  * **데이터 유효성 검증**: 마이그레이션 과정에서 데이터의 유효성을 검사합니다. 예를 들어, 나이스페이 결제 건임에도 PG사 거래 ID가 없는 데이터는 유효하지 않다고 판단하여 마이그레이션 대상에서 제외했습니다.

    ```python
    def check_if_nicepay_data_invalid(self, payment):
        """Check if data is invalid for nicepay method."""

        if payment["method"] == "nicepay":
            try:
                if not (
                    payment["data"]["tid"] is not None
                    and payment["data"]["timestamp"] is not None
                ):
                    raise NicepayDataInvalidError
            except (KeyError, TypeError, PaymentValidationError):
                self.log.info(
                    f"→ Skipped because data({payment['data']}) is invalid for nicepay method."
                )
                raise NicepayDataInvalidError
    ```
    {: .nolineno }

  * **리포트 생성**: 각 회차별 마이그레이션 결과를 리포트로 생성하여, 어떤 데이터가 성공적으로 이전되었고 어떤 데이터가 제외되었는지(그리고 그 사유는 무엇인지) 명확히 추적할 수 있도록 했습니다.
    * 마이그레이션 리포트 예
        ```json
        {
          "source": {
            "payment": {
              "count": {
                "_error": 0,
                "_warning": 0,
                "created": 18,
                "deleted": 0,
                "skipped": 0,
                "updated": 0
              },
              "id": {
                "_error": [],
                "_warning": [],
                "created": [
                  2579367,
                  2579368,
                  2579369,
                  ...
                ],
                "deleted": [],
                "first": 2579367,
                "last": 2579386,
                "skipped": [],
                "updated": []
              }
            }
          },
          "target": {
            "billing_key": {
              "count": {
                "_error": 0,
                "_warning": 0,
                "created": 8,
                "deleted": 0,
                "skipped": 0,
                "updated": 0
              },
              "id": {
                "_error": [],
                "_warning": [],
                "created": [
                  [
                    2579367,
                    288423
                  ],
                  [
                    2579369,
                    288424
                  ],
                  [
                    2579373,
                    288425
                  ],
                  ...
                ],
                "deleted": [],
                "skipped": [],
                "updated": []
              }
            },
            "payment": {
              "count": {
                "_error": 0,
                "_warning": 0,
                "created": 0,
                "deleted": 0,
                "skipped": 10,
                "updated": 0
              },
              "id": {
                "_error": [],
                "_warning": [],
                "created": [],
                "deleted": [],
                "skipped": [
                  [
                    "PaymentNotSucceededError",
                    2579368,
                    null
                  ],
                  [
                    "PaymentNotSucceededError",
                    2579370,
                    null
                  ],
                  [
                    "PaymentNotSucceededError",
                    2579374,
                    null
                  ],
                  ...
                ],
                "updated": []
              }
            },
            "payment_transaction": {
              "count": {
                "_error": 0,
                "_warning": 0,
                "created": 0,
                "deleted": 0,
                "skipped": 10,
                "updated": 0
              },
              "id": {
                "_error": [],
                "_warning": [],
                "created": [],
                "deleted": [],
                "skipped": [
                  [
                    "PaymentNotSucceededError",
                    2579368,
                    null
                  ],
                  [
                    "PaymentNotSucceededError",
                    2579370,
                    null
                  ],
                  [
                    "PaymentNotSucceededError",
                    2579374,
                    null
                  ],
                  ...
                ],
                "updated": []
              }
            },
            "user_config": {
              "count": {
                "_error": 0,
                "_warning": 0,
                "created": 8,
                "deleted": 0,
                "skipped": 0,
                "updated": 0
              },
              "id": {
                "_error": [],
                "_warning": [],
                "created": [
                  [
                    2579367,
                    326205
                  ],
                  [
                    2579369,
                    326206
                  ],
                  [
                    2579373,
                    326207
                  ],
                  ...
                ],
                "deleted": [],
                "skipped": [],
                "updated": []
              }
            }
          }
        }
        ```
        {: .nolineno }


### 결제 서비스 → 기존 모놀리식 서비스 마이그레이션

기존 모놀리식 서비스에는 제가 개발한 결제 서비스의 데이터를 이용하는 비즈니스 로직이 여전히 남아 있었습니다. 그래서 결제 서비스에서 생성된 결제 정보를 기존 모놀리식 서비스의 데이터베이스 테이블에 업데이트하는 작업이 필요했습니다. 이 과정은 결제 서비스에서 생성된 이벤트를 AWS SNS로 발행(publish)하고, AWS Lambda로 구독(subscribe)하여 기존 테이블을 업데이트하는 방식으로 처리했습니다.

그런데 기존 모놀리식 서비스와 신규 결제 서비스의 결제 정보 스펙이 서로 달랐기 때문에, 결제 서비스의 정보를 재구성해서 업데이트해야만 했습니다. 예를 들면 다음과 같은 규칙이 필요했습니다.

  * 결제 서비스의 자동 결제가 비활성화 상태이면, 기존 모놀리식 서비스의 결제 정보는 삭제합니다.
  * 결제 서비스에서 카드가 등록되더라도, 자동 결제 수단이 예치금으로 설정되어 있다면 기존 모놀리식 서비스의 결제 정보는 업데이트하지 않습니다.
  * 결제 서비스에서 기본 카드가 아닌 카드가 등록되거나 삭제되었을 때는 기존 모놀리식 서비스의 결제 정보를 업데이트하지 않습니다.

### 결제 전용 백오피스

![결제 백오피스 - 예치금 관리](/assets/posts/2020-12-01-payment-service/deposit-management.png)
_결제 백오피스 - 예치금 관리_

기존 백오피스의 결제 기능들은 대부분 모놀리식 서비스의 결제 모듈과 연동되어 있었기 때문에, 이를 새로운 결제 서비스의 API로 대체하는 작업이 필요했습니다. 하지만 서비스 구조가 전반적으로 변경되어 단순히 API만 교체하는 방식은 불가능했고, 몇 가지 대안이 나왔습니다.

1.  기존 백오피스에 새로운 위젯(widget)이나 모달(modal)을 추가하여 결제 서비스의 CRUD 기능을 모두 연동합니다.
2.  기존 백오피스에서는 주로 읽기(Read) 기능만 수행하고, 새로운 결제 전용 백오피스를 만들어 대부분의 결제 관련 작업을 처리합니다.

1번 방식은 결제 기능을 사용하기 위해 브라우저의 다른 탭으로 이동하지 않아도 되는 장점이 있었습니다. 그러나 해당 백오피스가 2016년에 처음 개발되어 대부분 React의 클래스형 컴포넌트로 작업되어 있었고, UI 컴포넌트 역시 Bootstrap과 Material UI를 사용하고 있었습니다.

결과적으로 2번 방식으로 결정하게 된 이유는 React의 함수형 컴포넌트와 Hooks를 사용하고 싶었고, UI 또한 점점 트렌드와 멀어지는 Bootstrap과 Material UI를 계속 사용하고 싶지 않았기 때문입니다. 마침 다른 시니어 개발자분께서도 "기존 백오피스 소스의 복잡도가 높으니, 전체 화면이 새로운 컴포넌트로 구성되는 페이지들은 Next.js를 사용해서 새로 만드는 게 좋겠다"는 의견을 주셨습니다.

기존 백오피스에서는 Redux로 상태 관리를 하고 있었는데, 당시에는 비동기 처리를 위해 Redux-Saga를 사용하는 것이 권장되는 추세였습니다. 그래서 처음에 React, Next.js, Redux-Saga 조합으로 보일러플레이트 코드를 구성했는데, 이를 본 사내 다른 개발자분들의 의견은 다음과 같았습니다.

  * 결제 서비스에 굳이 Redux-Saga를 사용할 필요는 없습니다. 동시에 여러 API를 요청해야 한다면 `Promise.all`을 사용하면 됩니다.
  * (아직 Redux-Saga를 사용한 프로젝트가 없으니) 다른 사람이 유지보수하기 어려운 모듈을 추가하지는 맙시다. 예를 들어, 현재는 퇴사한 분이 과거에 RxJS로 작업한 부분을 다른 동료들이 쉽게 이해하기 어려워합니다. 일단은 props를 사용하고, Context API도 좋은 대안이 될 수 있습니다.

저는 당시 React 경험이 거의 없었기 때문에 별다른 반박을 할 수 없었고, 동료들의 의견에 따라 기본적으로 props를 사용하되 상태 공유가 필요할 경우 Context API를 고려해보기로 했습니다.

UI 컴포넌트는 기능과 디자인을 고려하여 Blueprint, Tailwind UI, Ant Design을 최종 후보로 두었습니다. 처음에는 Blueprint의 테이블 컴포넌트로 작업을 시작했다가 렌더링 퍼포먼스가 기대에 미치지 못해 Ant Design으로 변경했는데, 기능과 디자인 모두 다른 라이브러리에 비해 압도적으로 만족스러웠습니다.

최종적으로 주문 정산 목록, 주문 정산 상세, 사용자 결제 방법 관리, 사용자 주문 관리, 사용자 예치금 관리, 이렇게 총 5가지의 화면을 작업했습니다. 이 중 기존 백오피스와 거의 비슷한 화면은 주문 정산 목록 1개뿐이었고, 나머지는 완전히 새로운 기능이거나 구조가 바뀐 화면들이었습니다. 사내에 파이낸스 담당 프로덕트 오너가 없어 와이어프레임 제작 같은 화면 기획도 온전히 저의 몫이었습니다. 결과적으로 혼자서 모든 부분을 담당하게 되었는데, 다음과 같은 장점들이 있었습니다.

  * 결제 서비스 API를 잘 이해하고 있었기 때문에 화면 기획을 효율적으로 할 수 있었습니다.
  * 화면에 대한 아이디어를 바로 구현해보고, 마음에 들지 않으면 즉각 수정할 수 있었습니다.
  * 프론트엔드 작업을 하다가 백엔드에 필요한 기능이 없을 때 API를 직접 업데이트하는 일이 잦았는데, 이 과정에서 발생하는 커뮤니케이션 비용을 크게 줄일 수 있었습니다.

백오피스 개발을 마무리하며 몇 가지 아쉬운 점은 다음과 같습니다.

  * 여러 화면에서 공통으로 사용하는 모달 등을 커스텀 훅(custom hook)으로 구현하지 않아 중복 코드가 일부 존재합니다.
  * 현재 특별한 문제 없이 사용 중이지만, 가독성 저하와 같은 단점들 때문에 `useCallback`이나 `React.Memo`를 사용한 렌더링 최적화는 아직 진행하지 못했습니다.

### 모델 인스턴스 Revision 관리

결제 서비스에서는 `django-reversion` 모듈을 사용해서 모델 인스턴스의 모든 변경 이력을 리비전(revision)으로 관리하고 있으며, 이를 통해 아래와 같은 처리가 가능합니다.

  * Django Admin 등에서 실수로 수정하거나 삭제한 레코드를 복구하고 싶을 때 사용할 수 있습니다.
  * 백오피스, 고객/파트너 앱을 통해 레코드를 생성하거나 수정한 사용자를 기록하고 싶을 때 유용합니다.
      * 예를 들어 어떤 고객의 자동 결제 설정이 실수로 비활성화되었다면, Django Admin에서 어느 관리자가 또는 어느 백오피스 사용자가 수정했는지 추적할 수 있습니다.
  * 레코드의 생성 및 수정과 관련된 전체 내역을 확인하고 싶을 때 조회할 수 있습니다.

![버전 목록](/assets/posts/2020-12-01-payment-service/django-reversion-list.png){: width="500" }
_버전 목록_

![각 버전 별 데이터, 복구 기능](/assets/posts/2020-12-01-payment-service/django-reversion-restore.png){: width="500" }
_각 버전 별 데이터, 복구 기능_

### 자동 정산 시트

이 작업을 시작했을 때, 반드시 해야 할 일은 기존 모놀리식 서비스의 주문 정보와 새로운 결제 서비스의 결제 정보를 조인(join)하는 쿼리를 만드는 것이었습니다. 당시에는 이 쿼리의 결과가 어떻게 정산에 사용되는지는 정확히 몰랐습니다. 호기심에 파이낸스팀에 여쭤보니, 매일 파트너에게 서비스 비용을 지급하기 위해 다음과 같은 절차를 거친다고 했습니다.

1.  매일 오전 10시 30분에 Redash를 이용해 서비스별 정산 쿼리 5\~7개를 각각 실행합니다. 요일에 따라 실행하는 쿼리가 조금씩 다릅니다.
2.  쿼리별로 결과를 받는 데 최대 1분 정도 소요됩니다. 모든 쿼리 실행이 완료되면 각 결과를 CSV 파일로 저장합니다.
3.  매년 혹은 매월 첫날에는 해당 연월의 폴더를 구글 드라이브에 생성합니다.
4.  어제 사용한 구글 시트 파일을 복사해서, 첫 번째 개요 시트의 날짜를 오늘 날짜로 수정하고 기존 데이터를 삭제합니다.
5.  서비스별 시트 5\~7개에 Redash에서 내려받은 CSV 파일을 각각 가져오기(import) 합니다.

이 이야기를 듣고 "그러면 오전 10시 30분에 구글 시트에 데이터가 자동으로 채워지게 만들어드릴까요?" 라고 제안드렸더니 당연히 좋다고 하셔서, 결과적으로 일의 양이 2배로 늘어나게 되었습니다. 구글 드라이브와 시트 API 사용법을 익히고 연동하는 데는 하루 정도 걸린 것 같습니다. 결과적으로 1번부터 5번까지의 모든 과정이 자동화되었고, 수작업으로 인해 발생할 수 있었던 사람의 실수도 원천적으로 방지하게 되었습니다.

이 자동 정산 시트 생성 기능은 Jenkins에서 Django의 API를 호출하는 방식으로 실행됩니다. 총 생성 시간이 1분 이상 소요되기 때문에 Celery 태스크(task)로 비동기 처리했습니다.

![Jenkins 자동 정산 시트 build 화면](/assets/posts/2020-12-01-payment-service/daily-partner-payment-sheet-generation-jenkins.png)
_Jenkins 자동 정산 시트 build 화면_

요일별 스케줄에 따라 전체 서비스의 데이터를 한 번에 취합하는 것이 기본 기능이며, Jenkins를 통해 평일 오전 10시 30분에 실행되도록 설정되었습니다. 또한, 특정 서비스의 데이터만 개별적으로 업데이트하고 싶을 때를 대비해 `PROCESS_TYPE`을 설정하여 해당 서비스의 데이터만 시트에 추가하는 기능도 만들었습니다. 이 기능은 기본 스케줄과 관계없이(요일 무관) 언제든 실행할 수 있습니다. 사실 처음에는 정책 변화에 유연하게 대응하기 위해 섬세하게 만든 기능이지만, 실제로 사용되지 않을 수도 있겠다고 생각했습니다. 그런데 최근 특정 서비스의 정산 스케줄을 화요일에서 목요일로 조정해달라는 요청이 왔습니다. 이런 경우 코드에 하드코딩된 스케줄을 직접 수정하지 않아도, 목요일에 개별 서비스 실행 기능을 이용해 데이터를 업데이트할 수 있습니다.

![자동 생성된 정산 시트 목록](/assets/posts/2020-12-01-payment-service/sheet-list.png)
_자동 생성된 정산 시트 목록_

