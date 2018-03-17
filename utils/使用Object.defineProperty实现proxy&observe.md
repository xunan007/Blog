```javascript
/**
 * use Object.defineProperty to realize data proxy & data observe
 **/

// data proxy
/**
 *
 * realize options._data.xxx --> target.xxx
 * @param {any} data
 * @param {any} target
 */
function proxy(data, target) {
    const that = target;
    Object.keys(data._data).forEach(key => {
        Object.defineProperty(that, key, {
            configurable: true,
            enumerable: true,
            get: function proxyGetter() {
                return data._data[key];
            },
            set: function proxyGetter(val) {
                data._data[key] = val;
            }
        });
    });
}
var options = {
    _data: {
        a: 1,
        b: 2,
        c: {
            d: 3
        }
    }
};
var target = {};
proxy(options, target);
```

```javascript
/**
 * realize data observable, use Object.defineProperty to transform data.
 * both array and object are transformed into getter and setter recursively.
 * @param {any} target
 */
function observe(target) {
    // just object could run defineReactive function.
    if (!target || typeof target !== 'object') {
        return false;
    }
    Object.keys(target).forEach(key => {
        defineReactive(target, key, target[key]);
    });
}
function defineReactive(data, key, val) {
    observe(val);
    Object.defineProperty(data, key, {
        get: function() {
            return val;
        },
        set: function(newVal) {
            console.log(`old value: ${val} --> new value: ${newVal}`);
            val = newVal;
        }
    });
}
var data = {
    a: 1,
    b: 2,
    c: {
        wawa: 'juju'
    }
};
observe(data);
```