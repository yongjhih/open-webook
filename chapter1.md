# 案例1 - 透過 Twitch 帳號建立一個 Twitch 行為統計網站

我們要有使用者的 Twitch 授權，並幫使用者建立帳號。

所以委託 oauth.io 取得 Twitch 授權，然後轉存 firebase 。

```html
<script src="//code.jquery.com/jquery-1.11.3.min.js"></script>
<script src="//code.jquery.com/jquery-migrate-1.2.1.min.js"></script>
<script src="oauth.min.js"></script>
<script src="//cdn.firebase.com/js/client/2.3.1/firebase.js"></script>

<script>
    $(document).ready(function() {
        $("#login").click(function(event) {
            OAuth.initialize('xxx')
            OAuth.redirect('twitch', "https://yongjhih.github.io/");
        });

        // ...

        OAuth.callback('twitch').done(function(result) {
            console.log(result);
        }).fail(function(e) {
        });
    });
</script>
```
