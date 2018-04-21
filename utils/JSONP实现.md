```javascript
(function (window, document) {
    /**
     * @param {String} url 请求的url
     * @param {Ojbect} opts 配置
     * @param {Function} cb 回调函数
     */
    var jsonp = function (url, opts, cb) {
        var id = opts.name || '_jp' + new Date().getTime();
        var timeout = opts.timeout || 10000;
        var param = opts.param || 'callback';
        var script = document.createElement('script');

        // 超时处理
        var timer = setTimeout(function () {
            cleanup();
            if (cb) {
                cb(new Error('timeout'));
            }
        }, timeout);
        // 清空操作
        function cleanup() {
            document.body.removeChild(script);
            window[id] = null;
            clearTimeout(timer);
        }
        // 函数赋值
        window[id] = function (data) {
            cleanup();
            cb(null, data);
        };

        url +=
            url.indexOf('?') === -1
                ? '?'
                : '&' + param + '=' + encodeURIComponent(id);

        script.src = url;
        document.body.appendChild(script);
    };
    window.$jsonp = jsonp;
})(window, document);
```