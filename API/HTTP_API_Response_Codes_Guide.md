# HTTP API Response Codes Guide

## Status Code Categories

| Range | Meaning | কখন ব্যবহার করবেন |
|---|---|---|
| **1xx** | Informational | Request processing শুরু হয়েছে, client-কে অপেক্ষা/continue করতে বলা হয়। খুব কম ব্যবহৃত। |
| **2xx** | Success | Request সফল হয়েছে। |
| **3xx** | Redirection | Resource অন্য URL-এ বা cache ব্যবহার করতে হবে। |
| **4xx** | Client Error | Client-এর request-এ সমস্যা। |
| **5xx** | Server Error | Server-এর অভ্যন্তরীণ সমস্যা। |

## 1xx Informational

| Code | Name | কখন ব্যবহার করবেন |
|---|---|---|
|100|Continue|বড় request upload করার আগে client-কে continue করতে বলা।|
|101|Switching Protocols|HTTP থেকে WebSocket ইত্যাদিতে protocol পরিবর্তন।|
|102|Processing|Long-running request এখনও processing হচ্ছে।|
|103|Early Hints|Browser-কে আগে থেকেই resource preload করতে hint দেয়।|

## 2xx Success

| Code | Name | কখন ব্যবহার করবেন |
|---|---|---|
|200|OK|GET সফল, অথবা PUT/PATCH সফল হলে data return করছেন।|
|201|Created|POST দিয়ে নতুন resource তৈরি হয়েছে।|
|202|Accepted|Request গ্রহণ করা হয়েছে, কিন্তু processing এখনও চলছে।|
|203|Non-Authoritative Information|Response modified/intermediate source থেকে এসেছে।|
|204|No Content|DELETE সফল, অথবা সফল operation কিন্তু body return করবেন না।|
|205|Reset Content|Client UI reset করতে বলা।|
|206|Partial Content|Range request-এর আংশিক data পাঠানো।|

## 3xx Redirection

| Code | Name | কখন ব্যবহার করবেন |
|---|---|---|
|300|Multiple Choices|একাধিক resource available।|
|301|Moved Permanently|Resource স্থায়ীভাবে নতুন URL-এ গেছে।|
|302|Found|সাময়িকভাবে অন্য URL-এ redirect।|
|303|See Other|POST-এর পর GET endpoint-এ redirect।|
|304|Not Modified|Browser cache ব্যবহার করতে বলা।|
|307|Temporary Redirect|Method পরিবর্তন না করে temporary redirect।|
|308|Permanent Redirect|Method পরিবর্তন না করে permanent redirect।|

## 4xx Client Errors

| Code | Name | কখন ব্যবহার করবেন |
|---|---|---|
|400|Bad Request|Request-এর format বা parameter ভুল।|
|401|Unauthorized|Authentication (যেমন JWT Token) নেই বা অবৈধ।|
|402|Payment Required|Reserved; payment-based API-তে ব্যবহার হতে পারে।|
|403|Forbidden|User authenticated, কিন্তু permission নেই।|
|404|Not Found|চাওয়া resource পাওয়া যায়নি।|
|405|Method Not Allowed|Endpoint সেই HTTP method সমর্থন করে না।|
|406|Not Acceptable|Requested response format server দিতে পারে না।|
|407|Proxy Authentication Required|Proxy authentication দরকার।|
|408|Request Timeout|Client request সম্পূর্ণ করতে দেরি করেছে।|
|409|Conflict|Duplicate resource বা state conflict (যেমন email already exists)।|
|410|Gone|Resource স্থায়ীভাবে সরিয়ে ফেলা হয়েছে।|
|411|Length Required|Content-Length header প্রয়োজন।|
|412|Precondition Failed|If-Match/If-Unmodified-Since condition fail।|
|413|Payload Too Large|Request body খুব বড়।|
|414|URI Too Long|URL খুব বড়।|
|415|Unsupported Media Type|ভুল Content-Type পাঠানো হয়েছে।|
|416|Range Not Satisfiable|Invalid range request।|
|417|Expectation Failed|Expect header satisfy করা যায়নি।|
|418|I'm a teapot|RFC joke status code।|
|421|Misdirected Request|ভুল server-এ request গেছে।|
|422|Unprocessable Entity|Data format ঠিক, কিন্তু validation fail করেছে।|
|423|Locked|Resource locked।|
|424|Failed Dependency|Dependent operation fail করেছে।|
|425|Too Early|Request খুব তাড়াতাড়ি এসেছে।|
|426|Upgrade Required|নতুন protocol দরকার।|
|428|Precondition Required|Precondition header প্রয়োজন।|
|429|Too Many Requests|Rate limit অতিক্রম হয়েছে।|
|431|Request Header Fields Too Large|Header খুব বড়।|
|451|Unavailable For Legal Reasons|আইনি কারণে resource unavailable।|

## 5xx Server Errors

| Code | Name | কখন ব্যবহার করবেন |
|---|---|---|
|500|Internal Server Error|Server-side unexpected error।|
|501|Not Implemented|Feature implement করা হয়নি।|
|502|Bad Gateway|Gateway upstream server থেকে invalid response পেয়েছে।|
|503|Service Unavailable|Server maintenance বা সাময়িকভাবে unavailable।|
|504|Gateway Timeout|Gateway upstream response-এর জন্য timeout হয়েছে।|
|505|HTTP Version Not Supported|HTTP version support করে না।|
|507|Insufficient Storage|Storage যথেষ্ট নয়।|
|508|Loop Detected|Infinite loop detect হয়েছে।|
