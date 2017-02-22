# express-validator

[![npm version](https://img.shields.io/npm/v/express-validator.svg)](https://www.npmjs.com/package/express-validator)
[![Build Status](https://img.shields.io/travis/ctavan/express-validator.svg)](http://travis-ci.org/ctavan/express-validator)
[![Dependency Status](https://img.shields.io/david/ctavan/express-validator.svg)](https://david-dm.org/ctavan/express-validator)
[![Coverage Status](https://img.shields.io/coveralls/ctavan/express-validator.svg)](https://coveralls.io/github/ctavan/express-validator?branch=master)

An [express.js]( https://github.com/visionmedia/express ) middleware for
[node-validator]( https://github.com/chriso/validator.js ).

- [Installation](#installation)
- [Usage](#usage)
- [Middleware options](#middleware-options)
- [Validation](#validation)
- [Validation by schema](#validation-by-schema)
- [Validation result](#validation-result)
  + [Result API](#result-api)
  + [Deprecated API](#deprecated-api)
  + [String formatting for error messages](#string-formatting-for-error-messages)
  + [Per-validation messages](#per-validation-messages)
- [Optional input](#optional-input)
- [Sanitizer](#sanitizer)
- [Regex routes](#regex-routes)
- [TypeScript](#typescript)
- [Changelog](#changelog)
- [License](#license)

## Installation

```
npm install express-validator
```

## Usage

```javascript
var util = require('util'),
    bodyParser = require('body-parser'),
    express = require('express'),
    expressValidator = require('express-validator'),
    app = express();

app.use(bodyParser.json());
app.use(expressValidator([options])); // Dòng này phải ngay lập tức sau  middlewares bodyParser!

app.post('/:urlparam', function(req, res) {

  // VALIDATION
  // checkBody chỉ kiểm tra req.body; không có thông số req khác
  // Similarly checkParams chỉ kiểm tra các method http get req.params (URL params) and
  // checkQuery chỉ kiểm tra req.query (GET params).
  
  req.checkBody('postparam', 'postparam không hợp lệ').notEmpty().isInt();
  req.checkParams('urlparam', 'urlparam không hợp lệ').isAlpha();
  req.checkQuery('getparam', 'getparam không hợp lệ').isInt();

  // .notEmpty().isInt() là các method bạn sẽ cấu hình để kiểm tra ==> return kiểu boolean
  
  // OR assert có thể được sử dụng để kiểm tra trên tất cả 3 loại params.
  // req.assert('postparam', 'postparam không hợp lệ').notEmpty().isInt();
  // req.assert('urlparam', 'urlparam không hợp lệ').isAlpha();
  // req.assert('getparam', 'getparam không hợp lệ').isInt();

  // SANITIZATION
  // as with validation này sẽ chỉ xác nhận tương ứng với kết quả
  // request object lấy kết quả 
  req.sanitizeBody('postparam').toBoolean();
  req.sanitizeParams('urlparam').toBoolean();
  req.sanitizeQuery('getparam').toBoolean();

  // OR find các param có liên quan trong mọi lĩnh vực
  req.sanitize('postparam').toBoolean();

  // Hoặc sử dụng `var result = yield req.getValidationResult();`
  // khi sử dụng generators e.g. với express
  // lấy kết quả  và hiển thị lỗi
  req.getValidationResult().then(function(result) {
    if (!result.isEmpty()) {
      res.status(400).send('There have been validation errors: ' + util.inspect(result.array()));
      return;
    }
    res.json({
      urlparam: req.params.urlparam,
      getparam: req.params.getparam,
      postparam: req.params.postparam
    });
  });
});

app.listen(8888);
```

// từ url tương ứng mà kết quả sẽ dẫn đến.

```
$ curl -d 'postparam=1' http://localhost:8888/test?getparam=1
{"urlparam":"test","getparam":"1","postparam":true}

$ curl -d 'postparam=1' http://localhost:8888/t1est?getparam=1
Đã có lỗi xác nhận: [
  { param: 'urlparam', msg: 'urlparam không hợp lệ', value: 't1est' } ]

$ curl -d 'postparam=1' http://localhost:8888/t1est?getparam=1ab
There have been validation errors: [
  { param: 'getparam', msg: 'getparam không hợp lệ', value: '1ab' },
  { param: 'urlparam', msg: 'getparam không hợp lệ', value: 't1est' } ]

$ curl http://localhost:8888/test?getparam=1&postparam=1
There have been validation errors: [
  { param: 'postparam', msg: 'postparam không hợp lệ', value: undefined} ]
```

## Middleware Options
#### `errorFormatter`
_function(param,msg,value)_

The `errorFormatter` tùy chọn có thể được sử dụng để chỉ định một chức năng mà xây dựng các đối tượng lỗi được sử dụng trong các kết quả xác nhận trở lại bởi `req.getValidationResult()`.<br>
Nó sẽ trả về một `Object` cái đó có `param`, `msg`, và  `value` keys được định nghĩa.

Nó là một customer lỗi mà sẽ trả lại các thông báo lỗi.

```javascript
//  Trong ví dụ này, giá trị formParam sẽ được biến thành dạng body  hữu ích cho việc hiển thị. 

app.use(expressValidator({
  errorFormatter: function(param, msg, value) {
      var namespace = param.split('.')
      , root    = namespace.shift()
      , formParam = root;

    while(namespace.length) {
      formParam += '[' + namespace.shift() + ']';
    }
    return {
      param : formParam,
      msg   : msg,
      value : value
    };
  }
}));
```

####`customValidators`
_{ "validatorName": function(value, [additional arguments]), ... }_

// validatorName == tên method ví dụ isArray()
// value là mặc định  
// có thể add thêm các đối số để so sánh additional arguments
 
Tùy chỉnh một phương pháp để định nghĩa lỗi

Các `customValidators` tùy chọn có thể được sử dụng để thêm phương pháp validator của bạn.

Tùy chọn này nên là một `Object` định nghĩa tên của validator và các logic liên quan. 


Xác định custom validators của bạn :

```javascript
app.use(expressValidator({
 customValidators: {
    isArray: function(value) {
        return Array.isArray(value);
    },
    gte: function(param, num) {
        return param >= num;
    }
 }
}));
```

Sử dụng chúng với tên method validator name:
```javascript
req.checkBody('users', 'Users must be an array').isArray();
req.checkQuery('time', 'Time must be an integer great than or equal to 5').isInt().gte(5)
```
####`customSanitizers`
_{ "sanitizerName": function(value, [additional arguments]), ... }_

The `customSanitizers` tùy chọn này có thể được sử dụng để thay thế giá trị đưa vào. 

Nó là một object và có tên sanitizer name mà bạn đặt khi bạn sử dụng nó sẽ chuyển giá trị ban đầu sang một new value mới.

 Cái này là để làm sạch dữ liệu . Ví dụ loại bỏ html code hay javascript code trong input 

Định nghĩa method của bạn custom sanitizers:

```javascript
app.use(expressValidator({
 customSanitizers: {
    toSanitizeSomehow: function(value) {
        var newValue = value;//some operations
        return newValue;
    },
 }
}));
```
Sau đó bạn có thể sử dụng với tên sanitizer name:
```javascript
req.sanitize('address').toSanitizeSomehow();
```

## Validation

#### req.check();
```javascript
   req.check('testparam', 'Error Message').notEmpty().isInt();
   req.check('testparam.child', 'Error Message').isInt(); // Tìm params theo cấu trúc /params/params lồng nhau
   req.check(['testparam', 'child'], 'Error Message').isInt(); // find nested params
```

Bắt đầu xác nhận của tham số. Nó sẽ tìm kiếm các params theo thứ tự `params` => http get.
`query` ==> http get query
`body` ==> http post method

Sau đó nó sẽ xác nhận validate. Bạn cần sử dụng 
 
Starts the validation of the specife parameter, will look for the parameter in `req` in the order `params`, `query`, `body`, then validate, Bạn cần sử dụng các kí hiệu "." hoặc một mảng array để truy cập các giá trị lồng nhau. 

Nếu 1 validator nhận được trong params, bạn sẽ gọi nó như thế `req.assert('reqParam').contains('thisString');`.

Validators được  thêm vào và  mối quan hệ. Xem [chriso/validator.js](https://github.com/chriso/validator.js) có sẵn " available validators" cho trình xác nhận validators, hoặc [add thêm vào ](#customvalidators).

#### req.assert();
Sử dụng giống như  [req.check()](#reqcheck).

#### req.validate();
Sử dụng giống như  [req.check()](#reqcheck).

#### req.checkBody();
Sử dụng giống như  [req.check()](#reqcheck), Chỉ áp dụng với `req.body`.

#### req.checkQuery();
Sử dụng giống như [req.check()](#reqcheck),  Chỉ áp dụng với  `req.query`.

#### req.checkParams();
Sử dụng giống như [req.check()](#reqcheck), Chỉ áp dụng với `req.params`.

#### req.checkHeaders();
Sử dụng giống như `req.headers`. Phương pháp này không được đảm bảo bởi`req.check()`.

#### req.checkCookies();
chỉ kiểm tra `req.cookies`. Phương pháp này không được đảm bảo `req.check()`.

## Validation by Schema

Bạn có thể check nhiều thứ cùng 1 lúc sử dụng sơ đồ đơn giản. 

Bạn có thể thông qua tin nhắn mỗi validator error với key `errorMessage` .
optional Validator có thể được chuyển qua key `options` như là một mảng khi các value khác nhau là required, 
hoặc là một giá trị null .

```javascript
req.checkBody({
 'email': {
    optional: {
      options: { checkFalsy: true } // hoặc : [{ checkFalsy: true }]
    },
    isEmail: {
      errorMessage: 'Email không hợp lệ'
    }
  },
  'password': {
    notEmpty: true,
    matches: {
      options: ['example', 'i'] // tùy chọn cho các validator với các tùy chọn thuộc tính giống như mảng array
      // options: [/example/i] // Giống như mảng tham số với các đối số url
    },
    errorMessage: 'Mật khẩu không hợp lệ' // Thông báo lỗi cho tham số
  },
  'name.first': { //
    optional: true, // Sẽ không xác nhận nếu trường có giá trị là rỗng.
    isLength: {
      options: [{ min: 2, max: 10 }],
      errorMessage: 'Must be between 2 and 10 chars long' // Error message for the validator, takes precedent over parameter message
    },
    errorMessage: 'Invalid First Name'
  }
});
```

Bạn cũng có thể xác định vị trí cụ thể thông qua add `in` thông số như hình dưới đây. ==> email trong query



```javascript
req.check({
 'email': {
    in: 'query',
    notEmpty: true,
    isEmail: {
      errorMessage: 'Invalid Email'
    }
  }
});
```
Trong các thuộc tính in luôn được ưu tiên cao nhất. phương pháp này bạn sử dụng trong `in:'query'` vì thế checkQuery() sẽ gọi phía trong. thâm chí bạn đang dùng `checkParams()` or `checkBody()`. Xem in có chứa những gì xem phía dưới có giải thích.


```javascript
var schema = {
 'email': {
    in: 'query',
    notEmpty: true,
    isEmail: {
      errorMessage: 'Invalid Email'
    }
  },
  'password': {
    notEmpty: true,
    matches: {
      options: ['example', 'i'] // xác nhận với các tài sản trong cùng 1 mảng
      // options: [/example/i] // matches also accepts the full expression in the first parameter
    },
    errorMessage: 'Invalid Password' // Error message for the parameter
  }
};

req.check(schema);        // will check 'password' no matter where it is but 'email' in query params
req.checkQuery(schema);   // will check 'password' and 'email' in query params
req.checkBody(schema);    // will check 'password' in body but 'email' in query params
req.checkParams(schema);  // will check 'password' in path params but 'email' in query params
req.checkHeaders(schema);  // will check 'password' in headers but 'email' in query params
```
Các giá trị hiện thời được hỗ trợ  `'body', 'params', 'query', 'headers'`. Nếu bạn đưa vào một in vị trí ko được hỗ trợ quá trình cho tham số hiện tại được bỏ qua.


## Validation result

Kết quả của validator. 

### Result API
Phương pháp  `req.getValidationResult()` trả về một  Promise với resolves tới một object kết quả.

```js
req.assert('email', 'required').notEmpty();
req.assert('email', 'valid email required').isEmail();
req.assert('password', '6 to 20 characters required').len(6, 20);

req.getValidationResult().then(function(result) {
  // do something with the validation result
});
```

Các API cho các đối tượng "result" kết quả là :


#### `result.isEmpty()`
Trả về một kiểu boolean xác định liệu có lỗi hay không.

#### `result.useFirstErrorOnly()`
Thiết lập các `firstErrorOnly` của result object, mà sửa đổi các `result.array()` và `result.mapped()` làm việc.<br>

Phương pháp này là thể kết nối, vì vậy sau đây là OK:

```js
result.useFirstErrorOnly().array();
```

#### `result.array()`
Trả về một mảng của các lỗi. <br>
Tất cả các lỗi cho tất cả các tham số hợp lệ sẽ được bao gồm, trừ khi bạn chỉ định rằng bạn muốn chỉ có lỗi đầu tiên của mỗi param bằng cách gọi `result.useFirstErrorOnly()`.

```javascript
var errors = result.array();

// errors will now contain something like this:
[
  {param: "email", msg: "required", value: "<received input>"},
  {param: "email", msg: "valid email required", value: "<received input>"},
  {param: "password", msg: "6 to 20 characters required", value: "<received input>"}
]
```

#### `result.mapped()`
Trả về một đối tượng của các lỗi, mà chính là tên tham số, và giá trị là một đối tượng lỗi như trả về bởi các định dạng lỗi.

Vì lý do lịch sử, theo mặc định phương pháp này sẽ trở lại các lỗi cuối cùng của mỗi tham số. <br>
Bạn có thể thay đổi hành vi này bằng cách gọi 'result.useFirstErrorOnly () `, vì vậy các lỗi đầu tiên lại được thay thế.

```javascript
var errors = result.mapped();

// errors will now be similar to this:
{
  email: {
    param: "email",
    msg: "valid email required",
    value: "<received input>"
  },
  password: {
    param: "password",
    msg: "6 to 20 characters required",
    value: "<received input>"
  }
}
```

#### `result.throw()`
Nếu có sai sót, ném một 'Error` đối tượng được trang trí với các API tương tự như các đối tượng kết quả xác thực. <br>
Hữu ích để đối phó với các lỗi xác nhận trong `khối catch` của một 'try..catch` hoặc promise.

```js
try {
  result.throw();
  res.send('success!');
} catch (e) {
  console.log(e.array());
  res.send('oops, validation failed!');
}
```

### Deprecated API
The following methods are deprecated.<br>
While they work, their API is unflexible and sometimes return weird results if compared to the bleeding edge `req.getValidationResult()`.

Additionally, these methods may be removed in a future version.

#### `req.validationErrors([mapped])`
Trả về lỗi đồng bộ trong các hình thức của một mảng, hoặc một đối tượng mà các bản đồ tham số lỗi trong trường hợp 'mapped' được thông qua như là `true`. <br> Nếu không có lỗi, giá trị trả về là `false`. `false`.

```js
var errors = req.validationErrors();
if (errors) {
  // do something with the errors
}
```

#### `req.asyncValidationErrors([mapped])`
Returns a promise that will either resolve if no validation errors happened, or reject with an errors array/mapping object. For reference on this, see `req.validationErrors()`.

```js
req.asyncValidationErrors().then(function() {
  // all good here
}, function(errors) {
  // damn, validation errors!
});
```

### String formatting for error messages

Error messages can be customized to include both the value provided by the user, as well as the value of any parameters passed to the validation function, using a standard string replacement format:

`%0` is replaced with user input
`%1` is replaced with the first parameter to the validator
`%2` is replaced with the second parameter to the validator
etc...

Example:
```javascript
req.assert('number', '%0 is not an integer').isInt();
req.assert('number', '%0 is not divisible by %1').isDivisibleBy(5);
```

*Note:* string replacement does **not** work with the `.withMessage()` syntax. If you'd like to have per-validator error messages with string formatting, please use the [Validation by Schema](#validation-by-schema) method instead.

### Per-validation messages

Xác nhận đơn giản  với `.withMessage()`. This can be chained with the rest of your validation, and if you don't use it for one of the validations then it will fall back to the default.

```javascript
req.assert('email', 'Invalid email')
    .notEmpty().withMessage('Email is required')
    .isEmail();

req.getValidationResult()
   .then(function(result){
     console.log(result.array());
   });

```

prints:

```javascript
[
  {param: 'email', msg: 'Email is required', value: '<received input>'}
  {param: 'email', msg: 'Invalid Email', value: '<received input>'}
]
```

## Optional input

You can use the `optional()` method to skip validation. By default, it only skips validation if the key does not exist on the request object. If you want to skip validation based on the property being falsy (null, undefined, etc), you can pass in `{ checkFalsy: true }`.

```javascript
req.checkBody('email').optional().isEmail();
//if there is no error, req.body.email is either undefined or a valid mail.
```

## Sanitizer

#### req.sanitize();
```javascript

req.body.comment = 'a <span>comment</span>';
req.body.username = '   a user    ';

req.sanitize('comment').escape(); // returns 'a &lt;span&gt;comment&lt;/span&gt;'
req.sanitize('username').trim(); // returns 'a user'

console.log(req.body.comment); // 'a &lt;span&gt;comment&lt;/span&gt;'
console.log(req.body.username); // 'a user'

```

Sanitizes the specified parameter (using 'dot-notation' or array), the parameter will be updated to the sanitized result. Cannot be chained, and will return the result. See [chriso/validator.js](https://github.com/chriso/validator.js) for available sanitizers, or [add your own](#customsanitizers).

If a sanitizer takes in params, you would call it like `req.sanitize('reqParam').whitelist(['a', 'b', 'c']);`.

If the parameter is present in multiple places with the same name e.g. `req.params.comment` & `req.query.comment`, they will all be sanitized.

#### req.filter();
Alias for [req.sanitize()](#reqsanitize).

#### req.sanitizeBody();
Same as [req.sanitize()](#reqsanitize), but only looks in `req.body`.

#### req.sanitizeQuery();
Same as [req.sanitize()](#reqsanitize), but only looks in `req.query`.

#### req.sanitizeParams();
Same as [req.sanitize()](#reqsanitize), but only looks in `req.params`.

#### req.sanitizeHeaders();
Only sanitizes `req.headers`. This method is not covered by the general `req.sanitize()`.

#### req.sanitizeCookies();
Only sanitizes `req.cookies`. This method is not covered by the general `req.sanitize()`.

## Regex routes

Express allows you to define regex routes like:

```javascript
app.get(/\/test(\d+)/, function() {});
```

You can validate the extracted matches like this:

```javascript
req.assert(0, 'Not a three-digit integer.').len(3, 3).isInt();
```

## TypeScript
If you have been using this library with [TypeScript](http://www.typescriptlang.org/),
you must have been using the type definitions from [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/e2af6d0/express-validator/express-validator.d.ts).

However, as of v3.1.0, the type definitions are shipped with the library.
So please uninstall the typings from DT. Otherwise they may cause conflicts


## Changelog

Check the [GitHub Releases page](https://github.com/ctavan/express-validator/releases).

## License

Copyright (c) 2010 Chris O'Hara <cohara87@gmail.com>, MIT License
