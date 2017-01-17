---
title: Working with APIs the Pythonic Way
date: 2017-01-16 22:21:03
tags:
---

同外部服务交互式是任何现代系统的一部分。无论是否是付费服务，授权的，分析系统或者一个内部系统，它们之间都需要相互沟通。

文中我们将一步步的来实现一个与支付网关交互的模块。

<img src="https://cdn-images-1.medium.com/max/800/1*kjFgbdDWQujT6-yQ3P-pWA.jpeg">

## 外部服务

让我们从一个虚构的付款服务开始。

去充值信用卡，我们需要一个信用卡令牌、充值金额和客户端提供的唯一ID。

```
POST https://foo-payments.com/api/charge
{
    token: <string>,
    amount: <number>,
    uid: <string>,
}
```

如果充值成功可以从请求得到200的返回状态，一个事务ID和充值过期时间。

```
200 OK
{
    uid: <string>,
    amount: <number>,
    token: <string>,
    expiration: <string, isoformat>,
    transaction_id: <number>
}
```

如果充值失败我们会得到400的状态码和错误码，以及错误信息。

```
400 Bad Request
{
    uid: <string>,
    error: <number>,
    message: <string>
}
```

这里有两个错误码我们需要处理，1=拒绝访问，2=被盗

## 简单的实现

通常，我们从一个相对简单的实现开始。

```
# payments.py
 
import uuid
import requests
PAYMENT_GATEWAY_BASE_URL = 'https://gw.com/api'
PAYMENT_GATEWAY_TOKEN = 'topsecret'

def charge(
    amount,
    token,
    timeout=5,
):
    """Charge.

    amount (int):
        Amount in cents to charge.
    token (str):
        Credit card token.
    timeout (int):
        Timeout in seconds.

    Returns (dict):
        New payment information.
    """
    headers = {
        "Authorization": "Bearer " + PAYMENT_GATEWAY_TOKEN,
    }

    payload = {
        "token": token,
        "amount": amount,
        "uid": str(uuid.uuid4()),
    }

    response = requests.post(
        PAYMENT_GATEWAY_BASE_URL + '/charge',
        json=payload,
        headers=headers,
        timeout=timeout,
    )
    response.raise_for_status()
return response.json()
```

90%的开发者都止步于此，那么这样有什么问题呢？



## 错误处理

这里有两种错误情况我们需要处理：

- HTTP错误，例如连接错误，超时或者连接被拒绝
- 远程支付错误，如拒绝或者被盗卡

使用'requests'是一个内部的实现细节，模块的使用者不需要了解这些信息。

<em>为了提供完整的API我们需要传递错误</em>

让我们从定义自定义错误开始：

```
# errors.py

class Error(Exception):
    pass

class Unavailable(Error):
    pass

class PaymentGatewayError(Error):
    def __init__(self, code, message):
        self.code = code
        self.message = message
class Refused(PaymentGatewayError):
    pass
class Stolen(PaymentGatewayError):
    pass
```

我 [之前写过](https://medium.com/@hakibenita/bullet-proofing-django-models-c080739be4e#.4ju7vgl0t) 关于使用错误基类的好处。

我们来把错误处理和日志添加到函数中。

```
import logging
from . import errors

logger = logging.getLogger('payments')

def charge(
    amount,
    token,
    timeout=5,
):

    ...

    try:
        response = requests.post(
            PAYMENT_GATEWAY_BASE_URL + '/charge',
            json=payload,
            headers=headers,
            timeout=timeout,
        )
        response.raise_for_status()

    except (requests.ConnectionError, requests.Timeout) as e:
        raise errors.Unavailable() from e

    except requests.exceptions.HTTPError as e:
        if e.status_code == 400:
            error = e.response.json()
            code = error['code']
            message = error['message']

        if code == 1:
            raise errors.Refused(code, message) from e
        elif code == 2:
            raise errors.Stolen(code, message) from e
        else:
            raise errors.PaymentGatewayError(code, message) from e
        
        logger.exception("Payment service had internal error.")
        raise errors.Unavailable() from e
```

很棒！现在我们的函数不再抛出'requests'的异常。类似被盗卡或者拒绝充值这种重要的错误都被抛出为自定义错误。

## 定义响应

我们的函数返回了一个字典。字典是一个好的且具有扩展性的数据结构，但是当你有一个预定义字段集合时使用更明确的数据类型将达到更好的效果。

在任何面向对象编程的课程中你都能学到所有东西都是一个对象。尽管在JAVA的世界是对的，但是Python有一个更轻量级的解决方案，更适合我们的例子-命名元组。

命名元组就想它听起来那样，元组里的成员都有名字。你可以像类一样的使用但消耗更少的空间。

让我们为响应定义一个命名的元组：

```
from collections import namedtuple
ChargeResponse = namedtuple('ChargeResponse', [
    'uid',
    'amount',
    'token',
    'expiration',
    'transaction_id',
])
```

当充值成功，我们创建一个`ChargeResponse`对象

```
from datetime import datetime

...

def charge(
    amount,
    token,
    timeout=5,
):

    ...
    data = response.json()
    charge_response = ChargeResponse(
        uid=uuid.UID(data['uid']),
        amount=data['amount'],
        token=data['token'],
        expiration=datetime.strptime(data['expiration'], "%Y-%m-%dT%H:%M:%S.%f"),
        transaction_id=data['transaction_id'],
    )
return charge_response
```

现在我们的方法返回了一个`ChargeResponse`对象。其他处理可以更简单的被添加，比如造型和验证。

在我们想象的支付网关的例子中，我们将过期时间转换为了一个`datetime`对象。使用者不需要猜测远程服务所使用的时间格式。

通过使用自定义的`class`作为返回值我们简化了支付供应商的序列化格式。如果响应是一个XML,我们仍要返回字典吗?那就有点尴尬了。

## 使用Session

为了节省访问API的时间我们可以使用会话。请求会话使用一个内部的连接池。指向相同服务器的请求可以因此获益。我们也有机会添加一些有用的配置比如阻塞的cookies。

```
import http.cookiejar

# A shared requests session for payment requests.
class BlockAll(http.cookiejar.CookiePolicy):
    def set_ok(self, cookie, request):
        return False
payment_session = requests.Session()
payment_session.cookies.policy = BlockAll()
…
def charge(
    amount,
    token,
    timeout=5,
):
    ...
    response = payment_session.post(...)
    ...
```

## 更多的行为

任何外部的服务，比如一个支付服务，有更多的行为。

我们的方法第一部分要考虑授权，请求和HTTP错误。第二部分要处理协议错误和充值所需的特殊序列化。

第一部分是与所有行为相关，而第二部分只涉及充值。

让我们来拆分这些方法来重用第一部分。

```
import uuid
import logging
import requests
import http.cookiejar
from datetime import datetime

logger = logging.getLogger('payments')

class BlockAll(http.cookiejar.CookiePolicy):
    def set_ok(self, cookie, request):
        return False
payment_session = requests.Session()
payment_session.cookies.policy = BlockAll()

def make_payment_request(path, payload, timeout=5):
    """Make a request to the payment gateway.

    path (str):
        Path to post to.
    payload (object):
        JSON-serializable request payload.
    timeout (int):
        Timeout in seconds.

    Raises
        Unavailable
        requests.exceptions.HTTPError

    Returns (response)
    """
    headers = {
        "Authorization": "Bearer " + PAYMENT_GATEWAY_TOKEN,
    }

    try:
        response = payment_session.post(
            PAYMENT_GATEWAY_BASE_URL + path,
            json=payload,
            headers=headers,
            timeout=timeout,
        )
    except (requests.ConnectionError, requests.Timeout) as e:
        raise errors.Unavailable() from e

    response.raise_for_status()
    return response.json()

def charge(amount, token):
    """Charge credit card.

    amount (int):
        Amount to charge in cents.
    token (str):
        Credit card token.

    Raises
        Unavailable
        Refused
        Stolen
        PaymentGatewayError

    Returns (ChargeResponse)
    """
    try:
        response = make_payment_request('/charge', {
            'uid': str(uuid.uuid4()),
            'amount': amount,
            'token': token,
        })

    except requests.HTTPError as e:
        if e.status_code == 400:
            error = e.response.json()
            code = error['code']
            message = error['message']

        if code == 1:
            raise Refused(code, message) from e

        elif code == 2:
            raise Stolen(code, message) from e

        else:
            raise PaymentGatewayError(code, message) from e

        logger.exception("Payment service had internal error")
        raise errors.Unavailable() from e
    data = response.json()
    return ChargeResponse(
        uid=uuid.UID(data['uid']),
        amount=data['amount'],
        token=data[token'],
        expiration=datetime.strptime(data['expiration'], "%Y-%m-%dT%H:%M:%S.%f"),
        transaction_id=data['transaction_id'],
    )
```

<em>这就是全部的代码</em>

在传输、序列化、授权和请求处理之间都有清晰的界限。我们的顶层方法`charge`拥有了一个良好定义的接口。

添加新的行为我们可以定义一个新的返回类型，叫做`make_payment_request`并用相同的方式处理响应。

```
RefundResponse = namedtuple('RefundResponse', [
    'transaction_id',
    'refunded_transation_id',
])

def refund(transaction_id):
    """Refund charge transaction.

    transaction_id (str):
        Transaction id to refund.
    
    Raises:
        ...

    Return (RefundResponse)
    """
    try:
        response = make_payment_request('/refund', {
            'uid': str(uuid.uuid4()),
            'transaction_id': transaction_id,
        })
    except requests.HTTPError as e:
        # TODO: Handle refund remote errors
    data = response.json()
    return RefundResponse(
        'transaction_id': data['transaction_id'],
        'refunded_transation_id': data['refunded_transation_id'],
    )
```



## 测试

使用外部API的挑战就是你不能（或者至少不应该）在自动化测试中去访问他们。我想要专注于测试我们的支付模块的代码而不是整个模块。

幸运的是，我们的模块有简单的接口所以可以很容易模拟。

让我们来测试一个叫`charge_user_for_product`的方法：

```
# test.py

from unittest import TestCase
from unittest.mock import patch

from payment.payment import ChargeResponse
from payment import errors

def TestApp(TestCase):
    @mock.patch('payment.charge')
    def test_should_count_transactions(self, mock_charge):
        mock_charge.return_value = ChargeResponse(
            uid=’test-uid’,
            amount=1000,
            token=’test-token’,
            expiration=datetime.datetime(2017, 1, 1, 15, 30, 7),
            transaction_id=12345,
        )
        charge_user_for_product(user, product)
        self.assertEqual(user.approved_transactions, 1)
    @mock.patch('payment.charge')
    def test_should_suspend_user_if_stolen(self, mock_charge):
        mock_charge.side_effect = errors.Stolen
        charge_user_for_product(user, product)
        self.assertEqual(user.is_active, False)
```

很直接明了，不需要模拟API的响应。测试包含在我们自定义的数据结构中，能够完全掌控。

<em>NOTE</em>: 另一个测试服务的方式是完成2种实现: 真实的和模拟的。在测试中，使用模拟的实现。

这就是依赖注入如何工作的。`Django`没有使用`DI（依赖注入）`但使用了与后端相同的概念（email, cache, template 等）。比如你可以用测试后端来测试邮件，内存后端来测试缓存(*For example you can test emails in django by using a test backend, test caching by using in-memory backend, etc.*)

如果你有多个''真正的"后端，还会有其他的好处。

无论你选择模拟服务请求还是像上面的插图一样注入一个假的服务，你必须要有一个良好定义的接口。



## 总结

在我们的程序中可能希望使用外部的服务。我们想要实现一个模块和外部模块交互，并使它坚固、可扩展和可复用。

我们需要以下几个步骤：

1. <em>简洁的实现</em>——使用'requests'发起请求并返回`json`响应
2. <em>错误处理</em>——自定义错误来包装传输和远程服务错误。使用者与传输和实现细节没有任何关联。
3. <em>标准化返回值</em>——使用命名元组返回一个变现远程服务器返回的类`class`类型。使用者同样不需要清楚序列化使用的格式。
4. <em>添加会话</em>——节省请求的消耗并提供存放全局连接配置的空间。
5. <em>分离请求和行为</em>——请求部分是可重用并且可以被新行为简单集成的。
6. <em>测试</em>——模拟请求并使用自定义数据和错误代替返回值。



