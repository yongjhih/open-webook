# 案例1 - 透過 Twitch 帳號建立一個 Twitch 行為統計網站

我們要有使用者的 Twitch 授權，並幫使用者建立帳號。

所以委託 oauth.io 取得 Twitch 授權，然後轉存 firebase 。

```js
<script src="oauth.min.js"></script>
<script>
    $(document).ready(function() {
        $("#login").click(function(event) {
            OAuth.initialize('xxx')
            OAuth.redirect('twitch', "https://xxx.github.io/");
        }

        // ...

        OAuth.callback('twitch').done(function(result) {
            //result.access_token
        }).fail(function(e) {
        });
    });
</script>
```

