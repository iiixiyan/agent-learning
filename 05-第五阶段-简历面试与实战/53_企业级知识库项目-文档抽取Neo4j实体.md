# 企业级知识库项目：文档抽取 Neo4j知识图谱的实体

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈
> 本文基于参考文章「Agent面试亮点：基于Neo4j知识图谱的GraphRAG」整理，内容侧重点有调整

---

var __INLINE_SCRIPT__ = (function () {
    'use strict';

    var Device = {};
    function detect(ua) {
      var MQQBrowser = ua.match(/MQQBrowser\/(\d+\.\d+)/i);
      var MQQClient = ua.match(/QQ\/(\d+\.(\d+)\.(\d+)\.(\d+))/i) || ua.match(/V1_AND_SQ_([\d\.]+)/);
      var WeChat = ua.match(/MicroMessenger\/((\d+)\.(\d+))\.(\d+)/) || ua.match(/MicroMessenger\/((\d+)\.(\d+))/);
      var MacOS = ua.match(/Mac\sOS\sX\s(\d+[\.|_]\d+)/);
      var WinOS = ua.match(/Windows(\s+\w+)?\s+?(\d+\.\d+)/);
      var Linux = ua.match(/Linux\s/);
      var MiuiBrowser = ua.match(/MiuiBrowser\/(\d+\.\d+)/i);
      var M1 = ua.match(/MI-ONE/);
      var MIPAD = ua.match(/MI PAD/);
      var UC = ua.match(/UCBrowser\/(\d+\.\d+(\.\d+\.\d+)?)/) || ua.match(/\sUC\s/);
      var IEMobile = ua.match(/IEMobile(\/|\s+)(\d+\.\d+)/) || ua.match(/WPDesktop/);
      var ipod = ua.match(/(ipod).*\s([\d_]+)/i);
      var ipad = ua.match(/(ipad).*\s([\d_]+)/i);
      var iphone = ua.match(/(iphone)\sos\s([\d_]+)/i);
      var Chrome = ua.match(/Chrome\/(\d+\.\d+)/);
      var AndriodBrowser = ua.match(/Mozilla.*Linux.*Android.*AppleWebKit.*Mobile Safari/);
      var android = ua.match(/(android)\s([\d\.]+)/i);
      var harmony = ua.match(/(OpenHarmony)\s([\d\.]+)/i);
      Device.browser = Device.browser || {}, Device.os = Device.os || {};
      Device.os.type = -1;
      Device.os.unifiedPC = ua.match(/UnifiedPC/);
      Device.os.unifiedMac = /UnifiedPCMac/i.test(ua);
      Device.os.unifiedWindows = /UnifiedPCWindows/i.test(ua);
      if (window.ActiveXObject) {
        var vie = 6;
        (window.XMLHttpRequest || ua.indexOf('MSIE 7.0') > -1) && (vie = 7);
        (window.XDomainRequest || ua.indexOf('Trident/4.0') > -1) && (vie = 8);
        ua.indexOf('Trident/5.0') > -1 && (vie = 9);
        ua.indexOf('Trident/6.0') > -1 && (vie = 10);
        Device.browser.ie = true, Device.browser.version = vie;
      } else if (ua.indexOf('Trident/7.0') > -1) {
        Device.browser.ie = true, Device.browser.version = 11;
      }
      if (android) {
        Device.os.android = true;
        Device.os.version = android[2];
        Device.os.type = 2;
      }
      if (harmony) {
        Device.os.harmony = true;
        Device.os.version = harmony[2];
        Device.os.type = 42;
      }
      if (ipod) {
        Device.os.ios = Device.os.ipod = true;
        Device.os.version = ipod[2].replace(/_/g, '.');
      }
      if (ipad) {
        Device.os.ios = Device.os.ipad = true;
        Device.os.version = ipad[2].replace(/_/g, '.');
        Device.os.type = 13;
      }
      if (iphone) {
        Device.os.iphone = Device.os.ios = true;
        Device.os.version = iphone[2].replace(/_/g, '.');
        Device.os.type = 1;
      }
      if (WinOS) Device.os.windows = true, Device.os.version = WinOS[2], Device.os.type = 15;
      if (MacOS) Device.os.Mac = true, Device.os.version = MacOS[1], Device.os.type = 14;
      if (Linux) Device.os.Linux = true, Device.os.type = 33;
      if (ua.indexOf('lepad_hls') > 0) Device.os.LePad = true;
      if (MIPAD) Device.os.MIPAD = true;
      if (MQQBrowser) Device.browser.MQQ = true, Device.browser.version = MQQBrowser[1];
      if (MQQClient) Device.browser.MQQClient = true, Device.browser.version = MQQClient[1];
      if (WeChat) Device.browser.WeChat = true, Device.browser.mmversion = Device.browser.version = WeChat[1];
      if (MiuiBrowser) Device.browser.MIUI = true, Device.browser.version = MiuiBrowser[1];
      if (UC) Device.browser.UC = true, Device.browser.version = UC[1] || NaN;
      if (IEMobile) Device.browser.IEMobile = true, Device.browser.version = IEMobile[2];
      if (AndriodBrowser) {
        Device.browser.AndriodBrowser = true;
      }
      if (M1) {
        Device.browser.M1 = true;
      }
      if (Chrome) {
        Device.browser.Chrome = true, Device.browser.version = Chrome[1];
      }
      if (Device.os.windows) {
        if (typeof navigator.platform !== "undefined" && navigator.platform.toLowerCase() == "win64") {
          Device.os.win64 = true;
        } else {
          Device.os.win64 = false;
        }
      }
      if (Device.os.Mac || Device.os.windows || Device.os.Linux || Device.os.unifiedPC || /OpenHarmony/i.test(ua) && /pc/i.test(ua)) {
        Device.os.pc = true;
      }
      var osType = {
        iPad7: 'iPad; CPU OS 7',
        LePad: 'lepad_hls',
        XiaoMi: 'MI-ONE',
        SonyDTV: "SonyDTV",
        SamSung: 'SAMSUNG',
        HTC: 'HTC',
        VIVO: 'vivo'
      };
      for (var os in osType) {
        Device.os[os] = ua.indexOf(osType[os]) !== -1;
      }
      Device.os.phone = Device.os.phone || /windows phone/i.test(ua);
      Device.os.getNumVersion = function () {
        return parseFloat(Device.os.version);
      };
      Device.os.hasTouch = 'ontouchstart' in window;
      if (Device.os.hasTouch && Device.os.ios && Device.os.getNumVersion() = 3.0;
      };
      Device.browser.isCanOcx = function () {
        return !!Device.os.windows && (!!Device.browser.ie || Device.browser.isFFCanOcx() || !!Device.browser.webkit);
      };
      Device.browser.isNotIESupport = function () {
        return !!Device.os.windows && (!!Device.browser.webkit || Device.browser.isFFCanOcx());
      };
      Device.userAgent = {};
      Device.userAgent.browserVersion = Device.browser.version;
      Device.userAgent.osVersion = Device.os.version;
      if (Device.os.unifiedPC) {
        if (Device.os.unifiedWindows) Device.os.type = 37;else if (Device.os.unifiedMac) Device.os.type = 38;else Device.os.type = 39;
      }
      delete Device.userAgent.version;
    }
    detect(window.navigator.userAgent);
    function canSupportH5Video() {
      var ua = window.navigator.userAgent,
        m = null;
      if (!!Device.os.android) {
        if (Device.browser.MQQ && Device.browser.getNumVersion() >= 4.2) {
          return true;
        }
        if (ua.indexOf('MI2') != -1) {
          return true;
        }
        if (Device.os.version >= '4' && (m = ua.match(/MicroMessenger\/((\d+)\.(\d+))\.(\d+)/))) {
          if (parseFloat(m[1]) >= 4.2) {
            return true;
          }
        }
        if (Device.os.version >= '4.1') {
          return true;
        }
      }
      return false;
    }
    function canSupportVideoMp4() {
      var video = document.createElement('video');
      if (typeof video.canPlayType === 'function') {
        if (video.canPlayType('video/mp4; codecs="mp4v.20.8"') === 'probably') {
          return true;
        }
        if (video.canPlayType('video/mp4; codecs="avc1.42E01E"') === 'probably' || video.canPlayType('video/mp4; codecs="avc1.42E01E, mp4a.40.2"') === 'probably') {
          return true;
        }
      }
      return false;
    }
    function canSupportAutoPlay() {
      if (Device.os.ios && Device.os.getNumVersion()  1 && arguments[1] !== undefined ? arguments[1] : 0;
      var canEqual = arguments.length > 2 && arguments[2] !== undefined ? arguments[2] : false;
      var nowVersionStr = Device.os.version;
      if (!nowVersionStr) return false;
      var versionArr = version.split('.');
      var nowVersionArr = nowVersionStr.split('.');
      for (var i = 0; i  0) return vi > nvi;
        if (cp = o.length) return { done: true }; return { done: false, value: o[i++] }; }, e: function e(_e) { throw _e; }, f: F }; } throw new TypeError("Invalid attempt to iterate non-iterable instance.\nIn order to be iterable, non-array objects must have a [Symbol.iterator]() method."); } var normalCompletion = true, didErr = false, err; return { s: function s() { it = it.call(o); }, n: function n() { var step = it.next(); normalCompletion = step.done; return step; }, e: function e(_e2) { didErr = true; err = _e2; }, f: function f() { try { if (!normalCompletion && it["return"] != null) it["return"](); } finally { if (didErr) throw err; } } }; }
    function _unsupportedIterableToArray$2(o, minLen) { if (!o) return; if (typeof o === "string") return _arrayLikeToArray$2(o, minLen); var n = Object.prototype.toString.call(o).slice(8, -1); if (n === "Object" && o.constructor) n = o.constructor.name; if (n === "Map" || n === "Set") return Array.from(o); if (n === "Arguments" || /^(?:Ui|I)nt(?:8|16|32)(?:Clamped)?Array$/.test(n)) return _arrayLikeToArray$2(o, minLen); }
    function _arrayLikeToArray$2(arr, len) { if (len == null || len > arr.length) len = arr.length; for (var i = 0, arr2 = new Array(len); i  b;
      }
    };
    function cpVersion(ver, op, canEq, type) {
      var mmver = false;
      switch (type) {
        case 'mac':
          mmver = getMac();
          break;
        case 'windows':
          mmver = getWindowsVersionFormat();
          break;
        case 'wxwork':
          mmver = getWxWork();
          break;
        case 'mpapp':
          mmver = getMpApp();
          break;
        case 'unifiedpc':
          mmver = getUnifiedPcVer();
          break;
        default:
          mmver = get();
          break;
      }
      if (!mmver) {
        return;
      }
      var mmversion = mmver.split('.');
      var version = ver.split('.');
      if (!/\d+/g.test(mmversion[mmversion.length - 1])) {
        mmversion.pop();
      }
      for (var i = 0, len = Math.max(mmversion.length, version.length); i = 64 && parseInt(v) = hexNum;
      }
      return false;
    }
    var Mmversion = {
      get: get,
      getMac: getMac,
      getMacOS: getMacOS,
      getWindows: getWindows,
      getInner: getInner,
      getWxWork: getWxWork,
      getMpApp: getMpApp,
      cpVersion: cpVersion,
      eqVersion: eqVersion,
      gtVersion: gtVersion,
      ltVersion: ltVersion,
      getPlatform: getPlatform,
      getVersionNumber: getVersionNumber,
      isWp: is_wp,
      isIOS: is_ios,
      isAndroid: is_android$1,
      isHarmony: is_harmony,
      isHarmonyWechat: is_harmony && is_wechat && cpVersion('1.0.0', 1, true),
      isInMiniProgram: is_in_miniProgram,
      isWechat: is_wechat,
      isMac: is_mac,
      isWindows: is_windows,
      isLinux: is_linux,
      isMacWechat: is_mac_wechat,
      isWindowsWechat: is_windows_wechat,
      isWxWork: is_wx_work,
      isOnlyWechat: is_wechat && !is_wx_work,
      isMpapp: is_mpapp,
      isIPad: is_ipad,
      isGooglePlay: is_google_play,
      isPrefetch: is_prefetch,
      isDonutAPP: is_donut_app,
      compareHexVersion: compareHexVersion,
      isPcWechat: is_windows_wechat || is_mac_wechat,
      xwebVersion: xweb_version,
      isUnifiedPcWechat: is_unified_pc_wechat
    };

    function _typeof(obj) {
      "@babel/helpers - typeof";

      return _typeof = "function" == typeof Symbol && "symbol" == typeof Symbol.iterator ? function (obj) {
        return typeof obj;
      } : function (obj) {
        return obj && "function" == typeof Symbol && obj.constructor === Symbol && obj !== Symbol.prototype ? "symbol" : typeof obj;
      }, _typeof(obj);
    }

    function _arrayWithHoles(arr) {
      if (Array.isArray(arr)) return arr;
    }

    function _iterableToArrayLimit(arr, i) {
      var _i = null == arr ? null : "undefined" != typeof Symbol && arr[Symbol.iterator] || arr["@@iterator"];
      if (null != _i) {
        var _s,
          _e,
          _x,
          _r,
          _arr = [],
          _n = !0,
          _d = !1;
        try {
          if (_x = (_i = _i.call(arr)).next, 0 === i) {
            if (Object(_i) !== _i) return;
            _n = !1;
          } else for (; !(_n = (_s = _x.call(_i)).done) && (_arr.push(_s.value), _arr.length !== i); _n = !0);
        } catch (err) {
          _d = !0, _e = err;
        } finally {
          try {
            if (!_n && null != _i["return"] && (_r = _i["return"](), Object(_r) !== _r)) return;
          } finally {
            if (_d) throw _e;
          }
        }
        return _arr;
      }
    }

    function _arrayLikeToArray$1(arr, len) {
      if (len == null || len > arr.length) len = arr.length;
      for (var i = 0, arr2 = new Array(len); i = 0; --o) { var i = this.tryEntries[o], a = i.completion; if ("root" === i.tryLoc) return handle("end"); if (i.tryLoc = 0; --r) { var o = this.tryEntries[r]; if (o.tryLoc = 0; --e) { var r = this.tryEntries[e]; if (r.finallyLoc === t) return this.complete(r.completion, r.afterLoc), resetTryEntry(r), y; } }, "catch": function _catch(t) { for (var e = this.tryEntries.length - 1; e >= 0; --e) { var r = this.tryEntries[e]; if (r.tryLoc === t) { var n = r.completion; if ("throw" === n.type) { var o = n.arg; resetTryEntry(r); } return o; } } throw new Error("illegal catch attempt"); }, delegateYield: function delegateYield(e, r, n) { return this.delegate = { iterator: values(e), resultName: r, nextLoc: n }, "next" === this.method && (this.arg = t), y; } }, e; }

    var doc$1 = {};
    var isAcrossOrigin$1 = false;
    var notFoundedMPPageAction = [];
    var __moon_report$1 = window.__moon_report || function () {};
    var MOON_JSAPI_KEY_OFFSET = 8;
    try {
      doc$1 = top.window.document;
    } catch (e) {
      isAcrossOrigin$1 = true;
    }
    if (!window.JSAPIEventCallbackMap) {
      window.JSAPIEventCallbackMap = {};
    }
    function ready(onBridgeReady) {
      var bridgeReady = function bridgeReady() {
        try {
          if (onBridgeReady) {
            window.onBridgeReadyTime = window.onBridgeReadyTime || Date.now();
            onBridgeReady();
          }
        } catch (e) {
          __moon_report$1([{
            offset: MOON_JSAPI_KEY_OFFSET,
            log: 'ready',
            e: e
          }]);
          throw e;
        }
        window.jsapiReadyTime = Date.now();
      };
      if (!isAcrossOrigin$1 && (typeof top.window.WeixinJSBridge === 'undefined' || !top.window.WeixinJSBridge.invoke)) {
        if (doc$1.addEventListener) {
          doc$1.addEventListener('WeixinJSBridgeReady', bridgeReady, false);
        } else if (doc$1.attachEvent) {
          doc$1.attachEvent('WeixinJSBridgeReady', bridgeReady);
          doc$1.attachEvent('onWeixinJSBridgeReady', bridgeReady);
        }
      } else {
        bridgeReady();
      }
    }
    var invokeNotWaitA8key = ['notifyPageInfo', 'updatePageAuth'
    ];
    var checkNotFoundedInvoke = function checkNotFoundedInvoke(methodName, args) {
      if (methodName === 'handleMPPageAction' && (args === null || args === void 0 ? void 0 : args.action) && notFoundedMPPageAction.includes(args === null || args === void 0 ? void 0 : args.action)) {
        return true;
      }
      return false;
    };
    function invoke$1(_x, _x2, _x3) {
      return _invoke.apply(this, arguments);
    }
    function _invoke() {
      _invoke = _asyncToGenerator( _regeneratorRuntime$1().mark(function _callee(methodName, args, callback) {
        return _regeneratorRuntime$1().wrap(function _callee$(_context) {
          while (1) switch (_context.prev = _context.next) {
            case 0:
              if (!(window.__secPageAuthPromise && !window.__is_page_auth_ok__ && !invokeNotWaitA8key.includes(methodName))) {
                _context.next = 3;
                break;
              }
              _context.next = 3;
              return window.__secPageAuthPromise;
            case 3:
              ready(function () {
                if (isAcrossOrigin$1) return false;
                if (_typeof(top.window.WeixinJSBridge) !== 'object') {
                  alert('请在微信中打开此链接');
                  return false;
                }
                if (checkNotFoundedInvoke(methodName, args)) {
                  setTimeout(function () {
                    if (callback) {
                      callback.apply(window, [{
                        err_msg: "".concat(methodName, ":fail"),
                        err_desc: 'action isn\'t supported'
                      }]);
                    }
                  }, 0);
                } else {
                  top.window.WeixinJSBridge.invoke(methodName, args, function () {
                    try {
                      for (var _len2 = arguments.length, rets = new Array(_len2), _key2 = 0; _key2  ".concat(ret.err_msg) : '';
                      if (['handleMPPageAction', 'handleVideoAction', 'handleHaokanAction'].indexOf(methodName) !== -1) {
                        var action = (args === null || args === void 0 ? void 0 : args.action) || '';
                        console.info('[system]', "[jsapi] invoke->".concat(methodName, ", action->").concat(action).concat(errMsg));
                      } else {
                        console.info('[system]', "[jsapi] invoke->".concat(methodName).concat(errMsg));
                      }
                      if (methodName === 'handleMPPageAction' && (args === null || args === void 0 ? void 0 : args.action) && ((ret === null || ret === void 0 ? void 0 : ret.err_desc) === 'action isn\'t supported' || (ret === null || ret === void 0 ? void 0 : ret.err_msg) === 'handleMPPageAction:fail action is not supported')) {
                        notFoundedMPPageAction.push(args === null || args === void 0 ? void 0 : args.action);
                      }
                      if (callback) {
                        callback.apply(window, rets);
                      }
                    } catch (e) {
                      __moon_report$1([{
                        offset: MOON_JSAPI_KEY_OFFSET,
                        log: "invoke;methodName:".concat(methodName),
                        e: e
                      }]);
                      throw e;
                    }
                  });
                }
              });
            case 4:
            case "end":
              return _context.stop();
          }
        }, _callee);
      }));
      return _invoke.apply(this, arguments);
    }
    function call(_x4) {
      return _call.apply(this, arguments);
    }
    function _call() {
      _call = _asyncToGenerator( _regeneratorRuntime$1().mark(function _callee2(methodName) {
        return _regeneratorRuntime$1().wrap(function _callee2$(_context2) {
          while (1) switch (_context2.prev = _context2.next) {
            case 0:
              if (!(window.__secPageAuthPromise && !window.__is_page_auth_ok__)) {
                _context2.next = 3;
                break;
              }
              _context2.next = 3;
              return window.__secPageAuthPromise;
            case 3:
              ready(function () {
                if (isAcrossOrigin$1) return false;
                if (_typeof(top.window.WeixinJSBridge) !== 'object') {
                  return false;
                }
                try {
                  top.window.WeixinJSBridge.call(methodName);
                } catch (e) {
                  __moon_report$1([{
                    offset: MOON_JSAPI_KEY_OFFSET,
                    log: "call;methodName:".concat(methodName),
                    e: e
                  }]);
                  throw e;
                }
              });
            case 4:
            case "end":
              return _context2.stop();
          }
        }, _callee2);
      }));
      return _call.apply(this, arguments);
    }
    function on$1(eventName, callback) {
      ready(function () {
        if (isAcrossOrigin$1) return false;
        if (_typeof(top.window.WeixinJSBridge) !== 'object' || !top.window.WeixinJSBridge.on) {
          return false;
        }
        if (!window.JSAPIEventCallbackMap[eventName]) {
          window.JSAPIEventCallbackMap[eventName] = [];
        }
        window.JSAPIEventCallbackMap[eventName].push(callback);
        if (window.JSAPIEventCallbackMap[eventName].length > 1) {
          return false;
        }
        top.window.WeixinJSBridge.on(eventName, function () {
          try {
            for (var _len = arguments.length, rets = new Array(_len), _key = 0; _key  ".concat(ret.err_msg) : '';
            console.info('[system]', "[jsapi] event->".concat(eventName).concat(errMsg));
            if (window.JSAPIEventCallbackMap[eventName] && window.JSAPIEventCallbackMap[eventName].length) {
              var result;
              for (var i = 0; i = 0; i--) {
          if (window.JSAPIEventCallbackMap[eventName][i] === callback) {
            window.JSAPIEventCallbackMap[eventName].splice(i, 1);
            result = true;
          }
        }
        return result;
      });
    }
    var JSAPI = {
      ready: ready,
      invoke: invoke$1,
      call: call,
      on: on$1,
      remove: remove
    };

    function _toPrimitive(input, hint) {
      if (_typeof(input) !== "object" || input === null) return input;
      var prim = input[Symbol.toPrimitive];
      if (prim !== undefined) {
        var res = prim.call(input, hint || "default");
        if (_typeof(res) !== "object") return res;
        throw new TypeError("@@toPrimitive must return a primitive value.");
      }
      return (hint === "string" ? String : Number)(input);
    }

    function _toPropertyKey(arg) {
      var key = _toPrimitive(arg, "string");
      return _typeof(key) === "symbol" ? key : String(key);
    }

    function _defineProperty(obj, key, value) {
      key = _toPropertyKey(key);
      if (key in obj) {
        Object.defineProperty(obj, key, {
          value: value,
          enumerable: true,
          configurable: true,
          writable: true
        });
      } else {
        obj[key] = value;
      }
      return obj;
    }

    function _classCallCheck(instance, Constructor) {
      if (!(instance instanceof Constructor)) {
        throw new TypeError("Cannot call a class as a function");
      }
    }

    function _defineProperties(target, props) {
      for (var i = 0; i  d2.exp) return 1;
          return 0;
        });
        var memCnt = 0;
        for (var i = 0; i = size) break;
          var key = keys[i];
          memCnt += JSON.stringify(data[key]).length;
          delete data[key];
        }
        return data;
      },
      'clear-all': function clearAll() {
        localStorage.clear();
        return {};
      }
    };
    function formatLogMsg(str) {
      return "[WXLS] ".concat(str);
    }

    var LS = function () {
      function LS(func, evictionPolicy, logger) {
        _classCallCheck(this, LS);
        this.logger = function () {};
        if (!func) throw 'require function name.';
        this.evictionPolicy = 'noeviction';
        this.key = func;
        if (typeof logger === 'function') {
          this.logger = function (str, type) {
            return logger(formatLogMsg(str), type);
          };
        }
        if (evictionPolicy && Object.keys(evictionPolicies).indexOf(evictionPolicy) !== -1) {
          this.evictionPolicy = evictionPolicy;
        }
        this.init();
      }
      _createClass(LS, [{
        key: "init",
        value: function init() {
          var _a, _b;
          this.check();
          if (Math.random() * 1000  now) {
              temp[key] = val;
            }
          }
          this.logger("check info: isReturn:".concat(isReturn, " data:").concat(JSON.stringify(temp)), 'info');
          if (isReturn) return temp;
          LS.setItem(this.key, JSON.stringify(temp), this.logger);
        }
      }, {
        key: "set",
        value: function set(key, val, exp) {
          var _a, _b;
          var data = this.check(true);
          data[key] = {
            val: val,
            exp: exp || +new Date()
          };
          try {
            if (localStorage.getItem(prefix + this.key)) localStorage.removeItem(prefix + this.key);
            localStorage.setItem(prefix + this.key, JSON.stringify(data));
            this.logger("first set success: LSlen:".concat((_a = window === null || window === void 0 ? void 0 : window.localStorage) === null || _a === void 0 ? void 0 : _a.length, " key:").concat(prefix + this.key, " data:").concat(JSON.stringify(data)), 'success');
          } catch (e) {
            this.logger("first set error: LSlen:".concat((_b = window === null || window === void 0 ? void 0 : window.localStorage) === null || _b === void 0 ? void 0 : _b.length, " error:").concat(e, " key:").concat(prefix + this.key, " data:").concat(JSON.stringify(data), " k:").concat(key, " v:").concat(val, " exp:").concat(exp), 'error');
            localStorage.clear();
            LS.setItem(this.key, JSON.stringify(_defineProperty({}, key, {
              val: val,
              exp: exp || +new Date()
            })), this.logger);
          }
        }
      }, {
        key: "get",
        value: function get(key) {
          var data = this.getData();
          data = data[key];
          return data ? data.val || null : null;
        }
      }, {
        key: "remove",
        value: function remove(key) {
          var data = this.getData();
          if (data[key]) delete data[key];
          LS.setItem(this.key, JSON.stringify(data), this.logger);
        }
      }], [{
        key: "getItem",
        value: function getItem(key) {
          key = prefix + key;
          return localStorage.getItem(key);
        }
      }, {
        key: "setItem",
        value: function setItem(key, val, logger) {
          var _a, _b;
          key = prefix + key;
          var n = 3;
          while (n--) {
            try {
              if (localStorage.getItem(key)) localStorage.removeItem(key);
              localStorage.setItem(key, val);
              typeof logger === 'function' && logger("setItem success: LSlen:".concat((_a = window === null || window === void 0 ? void 0 : window.localStorage) === null || _a === void 0 ? void 0 : _a.length, " key:").concat(key, " val:").concat(val), 'success');
              break;
            } catch (e) {
              typeof logger === 'function' && logger("setItem error: LSlen:".concat((_b = window === null || window === void 0 ? void 0 : window.localStorage) === null || _b === void 0 ? void 0 : _b.length, " error:").concat(e, " key:").concat(key, " val:").concat(val), 'error');
              LS.clear();
            }
          }
        }
      }, {
        key: "clear",
        value: function clear() {
          var i;
          var k;
          for (i = localStorage.length - 1; i >= 0; i--) {
            k = localStorage.key(i);
            if (k.indexOf(prefix) == 0) {
              localStorage.removeItem(k);
            }
          }
        }
      }, {
        key: "getSupportEvicationPolicy",
        value: function getSupportEvicationPolicy() {
          return Object.keys(evictionPolicies);
        }
      }]);
      return LS;
    }();
    var innerVersion = (Mmversion.getInner() || '').toUpperCase();
    var getBizLS = new LS('get_biz_result');
    function getBizMap() {
      if (!window.__get_biz_map__) {
        window.__get_biz_map__ = {};
      }
      return window.__get_biz_map__;
    }
    var isGetBizSupported = Mmversion.isOnlyWechat && Mmversion.isIOS && innerVersion >= '18003C2A' || Mmversion.isOnlyWechat && Mmversion.isAndroid && innerVersion >= '28003D3C' || Mmversion.isUnifiedPcWechat && Mmversion.cpVersion('4.1.10', 1, true, 'unifiedpc');
    function invokeGetBiz(needCheckBiz, bizType) {
      return dedupePromise("getBiz:".concat(needCheckBiz, ":").concat(bizType), function () {
        return new Promise(function (resolve, reject) {
          if (!isGetBizSupported) {
            reject('Not support');
          } else {
            JSAPI.invoke('handleMPPageAction', {
              action: 'getBiz',
              needCheckBiz: needCheckBiz,
              bizType: bizType
            }, function (res) {
              console.log("getBiz needCheckBiz=".concat(needCheckBiz, " bizType=").concat(bizType, " res: ").concat(JSON.stringify(res)));
              if (res && res.err_msg && res.err_msg.indexOf('ok') > -1) {
                var bizMap = getBizMap();
                bizMap[bizType] = res.biz;
                resolve(res.biz);
                getBizLS.set("".concat(bizType, "_get_biz_result"), res.biz, +new Date() + 3 * 24 * 60 * 60 * 1000);
              } else {
                reject('Failed to get biz');
              }
            });
          }
        });
      });
    }
    function getBiz(needCheckBiz, bizType) {
      var _a;
      if (needCheckBiz === void 0) {
        needCheckBiz = false;
      }
      if (bizType === void 0) {
        bizType = ((_a = window.cgiDataNew) === null || _a === void 0 ? void 0 : _a.biz_type) || 1;
      }
      var bizMap = getBizMap();
      if (!needCheckBiz && bizMap[bizType] !== undefined) {
        return Promise.resolve(bizMap[bizType]);
      }
      return invokeGetBiz(needCheckBiz, bizType);
    }
    Mmversion.isOnlyWechat && Mmversion.isIOS || Mmversion.isOnlyWechat && Mmversion.isAndroid || Mmversion.isUnifiedPcWechat && Mmversion.cpVersion('4.1.10', 1, true, 'unifiedpc');
    var getIsAuthor = function getIsAuthor(cb, bizuin, needCheckBiz, bizType) {
      var _a;
      if (bizuin === void 0) {
        bizuin = window.biz;
      }
      if (needCheckBiz === void 0) {
        needCheckBiz = false;
      }
      if (bizType === void 0) {
        bizType = ((_a = window.cgiDataNew) === null || _a === void 0 ? void 0 : _a.biz_type) || 1;
      }
      getBiz(needCheckBiz, bizType).then(function (biz) {
        cb(biz && biz === bizuin);
      })["catch"](function () {
        cb(false);
      });
    };

    function parseUrl(url) {
      var len = url.length;
      var ques_pos = url.indexOf('?');
      var hash_pos = url.indexOf('#');
      hash_pos = hash_pos == -1 ? len : hash_pos;
      ques_pos = ques_pos == -1 ? hash_pos : ques_pos;
      var host = url.substring(0, ques_pos);
      var query_str = url.substring(ques_pos + 1, hash_pos);
      var hash = url.substring(hash_pos + 1);
      return {
        host: host,
        query_str: query_str,
        hash: hash
      };
    }
    function join(url, args, noEncode) {
      var ret = parseUrl(url);
      var query_str = ret.query_str;
      var args_arr = [];
      if (_typeof(args) === 'object') {
        for (var key in args) {
          if (args.hasOwnProperty(key)) {
            args_arr.push("".concat(key, "=").concat(noEncode ? args[key] : encodeURIComponent(args[key])));
          }
        }
      } else {
        args_arr.push(noEncode ? args : encodeURIComponent(args));
      }
      if (args_arr.length > 0) {
        query_str += (query_str !== "" ? "&" : "") + args_arr.join("&");
      }
      return ret.host + (query_str !== "" ? "?".concat(query_str) : "") + (ret.hash !== "" ? "#".concat(ret.hash) : "");
    }

    function addParam(url, param, value, forceReplace) {
      url = url || location.href;
      var firstAndPos = url.indexOf("&");
      var len = url.length;
      var reverseUrl = url.replace(/^[\w\d]+:[/\\]+/g, "").split("").reverse();
      if (!Array.prototype.indexOf) {
        Array.prototype.indexOf = function (searchElement, fromIndex) {
          var k;
          if (this == null) {
            throw new TypeError('"this" is null or not defined');
          }
          var O = Object(this);
          var len = O.length >>> 0;
          if (len === 0) {
            return -1;
          }
          var n = fromIndex || 0;
          if (Math.abs(n) === Infinity) {
            n = 0;
          }
          if (n >= len) {
            return -1;
          }
          k = Math.max(n >= 0 ? n : len - Math.abs(n), 0);
          while (k  lastSlashPos) {
        url = url.replace("&", "?");
      }
      var reg = new RegExp("([\\?&]".concat(param, "=)[^&#]*"));
      if (!url.match(reg)) {
        var urlInfo = parseUrl(url);
        var hash = urlInfo.hash ? '#' + urlInfo.hash : '';
        url = url.replace(hash, '');
        var _pos = url.indexOf("?");
        if (_pos == -1) {
          return "".concat(url, "?").concat(param, "=").concat(value).concat(hash);
        }
        if (_pos == url.length - 1) {
          return "".concat(url + param, "=").concat(value).concat(hash);
        }
        return "".concat(url, "&").concat(param, "=").concat(value).concat(hash);
      }
      if (forceReplace === true) {
        return url.replace(reg, "$1".concat(value));
      }
      return url;
    }
    function addWxfrom(src, wxfrom) {
      var offset = window.service_type === 1 ? 10000 : 0;
      return addParam(src, 'wxfrom', offset + Number(wxfrom), true);
    }
    function removeParam(url, param) {
      var _URL = new URL(url),
        protocol = _URL.protocol,
        host = _URL.host,
        pathname = _URL.pathname,
        search = _URL.search,
        hash = _URL.hash;
      var queryParams = new URLSearchParams(search);
      queryParams["delete"](param);
      var newSearch = queryParams.toString();
      var newUrl = new URL("".concat(protocol, "//").concat(host).concat(pathname).concat(newSearch ? "?".concat(decodeURIComponent(newSearch)) : "").concat(hash));
      return newUrl.toString();
    }
    function getQuery(name, url) {
      var u = url || window.location.search;
      var reg = new RegExp("(^|&)".concat(name, "=([^&]*)(&|$)"));
      var r = u.substring(u.indexOf('?') + 1).match(reg);
      return r !== null ? r[2] : '';
    }
    function encodeBase64(value) {
      try {
        return window.btoa(value);
      } catch (e) {
        return '';
      }
    }
    function decodeBase64(value) {
      try {
        return window.atob(value);
      } catch (e) {
        return '';
      }
    }
    function joinUrl$1(url) {
      var obj = {};
      if (typeof window.uin !== 'undefined') {
        obj.uin = window.uin;
      }
      if (typeof window.key !== 'undefined') {
        obj.key = window.key;
      }
      if (typeof window.pass_ticket !== 'undefined') {
        obj.pass_ticket = window.pass_ticket;
      }
      if (typeof window.wxtoken !== 'undefined') {
        obj.wxtoken = window.wxtoken;
      }
      if (typeof window.devicetype !== 'undefined') {
        obj.devicetype = window.devicetype;
      }
      if (typeof window.clientversion !== 'undefined') {
        obj.clientversion = window.clientversion || Mmversion.getInner();
      }
      obj.version = obj.clientversion;
      if (window.biz) {
        obj.__biz = window.biz;
      }
      if (getQuery('enterid')) {
        obj.enterid = getQuery('enterid');
      }
      if (typeof window.appmsg_token !== 'undefined') {
        obj.appmsg_token = window.appmsg_token;
      } else if (url.indexOf('advertisement_report') > -1) {
        new Image().src = "".concat(location.protocol, "//mp.weixin.qq.com/mp/jsmonitor?idkey=68064_13_1&r=").concat(Math.random());
      }
      obj.x5 = navigator.userAgent.indexOf('TBS/') !== -1 ? '1' : '0';
      obj.f = 'json';
      return join(url, obj);
    }
    function joinUserArticleRole(url, notJoin, cb) {
      var bizuin = arguments.length > 3 && arguments[3] !== undefined ? arguments[3] : window.biz;
      var needCheckBiz = arguments.length > 5 && arguments[5] !== undefined ? arguments[5] : false;
      if (notJoin) {
        cb(url);
      } else {
        getIsAuthor(function (isAuthor) {
          cb(addParam(url, 'user_article_role', isAuthor ? 1 : 0, true));
        }, bizuin, needCheckBiz);
      }
    }
    function getA8keyQuery(name, url) {
      return new Promise(function (resolve) {
        if (window.__secPageAuthPromise) {
          window.__secPageAuthPromise.then(function () {
            resolve(getQuery(name, url));
          });
        } else {
          resolve(getQuery(name, url));
        }
      });
    }
    function addHash(url, hash) {
      var isReplace = arguments.length > 2 && arguments[2] !== undefined ? arguments[2] : false;
      if (isReplace) {
        return "".concat(url.split('#')[0]).concat(hash);
      }
      return "".concat(url).concat(url.indexOf('#') === -1 ? '#' : '').concat(hash);
    }
    function decodeUrl(url) {
      var _url = url;
      while (_url.indexOf('&amp;') !== -1) {
        _url = _url.htmlDecode();
      }
      return _url;
    }
    var Url = {
      parseUrl: parseUrl,
      join: join,
      addParam: addParam,
      addWxfrom: addWxfrom,
      addHash: addHash,
      getQuery: getQuery,
      getA8keyQuery: getA8keyQuery,
      encodeBase64: encodeBase64,
      decodeBase64: decodeBase64,
      joinUrl: joinUrl$1,
      joinUserArticleRole: joinUserArticleRole,
      removeParam: removeParam,
      decodeUrl: decodeUrl
    };

    Device.os.ipad && Device.os.getNumVersion() >= 13 && Device.os.getNumVersion() ".concat(jsapiName, " ").concat(opt.action || '', " ").concat(errMsg));
              typeof callback === 'function' && callback(ret);
            } catch (e) {
              window.WX_BJ_REPORT.BadJs.report('invoke', "callback ".concat(jsapiName, " error:"), {
                mid: 'mmbizwebapp:js_brridge',
                _info: e
              });
              console.error("[mpapp jsapi] ".concat(jsapiName, " ").concat(opt.action || ''), e, res);
            }
          });
        } catch (e) {
          window.WX_BJ_REPORT.BadJs.report('invoke', 'callback error:', {
            mid: 'mmbizwebapp:js_brridge',
            _info: e
          });
          console.error('[mpapp jsapi]', e);
        }
      });
    }

    function _log(level, msg) {
      if (level === 'log') {
        level = 'info';
        msg = "[WechatFe]".concat(msg);
      } else {
        var prefix = "__wap__".concat(window.__second_open__ ? ' (sec)' : '');
        msg = "".concat(prefix, " ").concat(msg, " location:[").concat(location.href, "]");
      }
      msg += new Error().stack;
      if (Mmversion.isMpapp) {
        invoke('WNNativeCallbackLog', msg);
      } else if (Mmversion.isWechat) {
        if (Mmversion.isAndroid) {
          console.warn('[system]', "[MicroMsg.JsApiLog][".concat(level, "] jslog : ").concat(msg));
        } else if (Mmversion.isIOS) {
          JSAPI.invoke('writeLog', {
            level: level,
            msg: msg
          });
        } else {
          JSAPI.invoke('log', {
            level: level,
            msg: msg
          });
        }
      }
    }
    var Log = {
      info: function info() {
        for (var _len = arguments.length, args = new Array(_len), _key = 0; _key = 0) continue;
        target[key] = source[key];
      }
      return target;
    }
    function formatDataToString(data) {
      var reportData = [];
      for (var key in data) {
        if (Object.prototype.hasOwnProperty.call(data, key)) {
          reportData.push(key + '=' + encodeURIComponent(data[key]));
        }
      }
      return reportData.join('&');
    }
    monitor.getReportData = function (opt) {
      opt = opt || {};
      var idkey = monitor._reportOptions.idkey || {};
      var key = null;
      var reportData = {};
      var nextKey;
      try {
        for (key in idkey) {
          if (Object.prototype.hasOwnProperty.call(idkey, key) && idkey[key]) {
            reportLogs.push(key + '_' + idkey[key]);
          }
        }
      } catch (e) {
        return false;
      }
      if (reportLogs.length === 0) {
        return false;
      }
      if (reportExtraLogs.length) {
        reportData.lc = reportExtraLogs.length;
        reportExtraLogs.forEach(function (extraLog, index) {
          reportData["log".concat(index)] = extraLog;
        });
      }
      try {
        var reportOptions = monitor._reportOptions;
        if (reportOptions !== null && reportOptions !== undefined) {
          for (nextKey in reportOptions) {
            if (Object.prototype.hasOwnProperty.call(reportOptions, nextKey)) {
              reportData[nextKey] = reportOptions[nextKey];
            }
          }
        }
      } catch (e) {
        reportData = {};
      }
      reportData.idkey = reportLogs.join(';');
      reportData.t = Math.random();
      if (opt.remove !== false) {
        reportLogs = [];
        reportExtraLogs = [];
        monitor._reportOptions = {
          idkey: {}
        };
      }
      return reportData;
    };
    monitor.setLogs = function (opt) {
      var id = opt.id;
      var key = opt.key;
      var value = opt.value;
      var extraLog = opt.log;
      var others = ObjWithoutProperty(opt, ['id', 'key', 'value', 'log']);
      var idkey = monitor._reportOptions.idkey || {};
      var param = id + '_' + key;
      if (idkey[param]) {
        idkey[param] += value;
      } else {
        idkey[param] = value;
      }
      monitor._reportOptions.idkey = idkey;
      if (extraLog) {
        reportExtraLogs.push(extraLog);
      }
      try {
        if (others !== null && others !== undefined) {
          for (var otherKey in others) {
            if (Object.prototype.hasOwnProperty.call(others, otherKey)) {
              monitor._reportOptions[otherKey] = others[otherKey];
            }
          }
        }
      } catch (e) {
        console.log(e);
      }
      return monitor;
    };
    monitor.setAvg = function (id, key, value) {
      var idkey = monitor._reportOptions.idkey || {};
      var param1 = id + '_' + key;
      var param2 = id + '_' + (key - 1);
      if (idkey[param1]) {
        idkey[param1] += value;
      } else {
        idkey[param1] = value;
      }
      if (idkey[param2]) {
        idkey[param2] += 1;
      } else {
        idkey[param2] = 1;
      }
      monitor._reportOptions.idkey = idkey;
      return monitor;
    };
    monitor.setSum = function (id, key) {
      var value = arguments.length > 2 && arguments[2] !== undefined ? arguments[2] : 1;
      var idkey = monitor._reportOptions.idkey;
      var param = id + '_' + key;
      if (idkey[param]) {
        idkey[param] += value;
      } else {
        idkey[param] = value;
      }
      monitor._reportOptions.idkey = idkey;
      return monitor;
    };
    monitor.send = function (async, ajax, origin) {
      if (async !== false) {
        async = true;
      }
      var data = monitor.getReportData();
      origin = origin || '';
      if (!data) {
        return;
      }
      if (!!ajax && ajax instanceof Function) {
        ajax({
          url: origin + sendUrl,
          type: 'POST',
          mayAbort: true,
          data: data,
          async: async,
          timeout: 2000,
          dontReport: true
        });
      } else {
        new Image().src = origin + '/mp/jsmonitor?' + formatDataToString(data) + '#wechat_redirect';
      }
    };
    if (typeof window !== 'undefined' && window.__monitor) {
      monitor = window.__monitor;
    } else {
      typeof window !== 'undefined' && (window.__monitor = monitor);
    }
    var monitor$1 = monitor;

    var logList = [];
    var log = function log(msg) {
      logList.push(msg);
    };
    var printLog = function printLog() {
      for (var i = 0, len = logList.length; i  [response ".concat(item.requestType, "]"), item.url, item.response, item);
      if (item.rdevRequestId && ((_b = (_a = window.RemoteDevSdk) === null || _a === void 0 ? void 0 : _a.instance) === null || _b === void 0 ? void 0 : _b.Network) && item.id !== '__system_log__') {
        try {
          var finishedOptions = {
            requestId: item.rdevRequestId,
            url: item.url,
            status: +(item.status || '500'),
            statusText: StatusTextMap[+(item.status || '500')] || 'Error',
            responseHeaders: {
              RDEV_RESPONSE_TYPE: item.requestType
            },
            responseBody: item.response,
            requestTime: item.requestTime || 0,
            duration: item.costTime || (item.endTime && item.startTime ? item.endTime - item.startTime : performance.now() / 1000 - (item.requestTime || 0))
          };
          window.RemoteDevSdk.instance.Network.customRequestFinished(finishedOptions);
        } catch (err) {}
      }
      if (((_c = window.vConsole) === null || _c === void 0 ? void 0 : _c.network) && item.id !== '__system_log__') {
        try {
          item.statusText = "".concat(item.status);
          item.responseSize = item.response.length;
          item.responseSizeText = "".concat(item.response.length);
          return (_e = (_d = window.vConsole.network).update) === null || _e === void 0 ? void 0 : _e.call(_d, item.id, Object.assign({}, item, {
            readyState: 4
          }));
        } catch (err) {}
      }
    }
    function reqType(obj, path) {
      return obj.url.indexOf(path) > -1 && obj.url.indexOf('action=') === -1 && (!obj.data || !obj.data.action);
    }
    function findAjaxScopeByConfig(url, config) {
      var pathname = new URL(url, location.href).pathname || '';
      var scope = config[pathname.slice(1)];
      if (scope) {
        return scope;
      }
    }
    function getAjaxScope(ajaxUrl) {
      if (Url.getQuery('no_transfer', location.href) !== '1' && Mmversion.isWechat && !Mmversion.isInMiniProgram && !Mmversion.isWxWork && !Mmversion.isMpapp && !isAcrossOrigin && window.__ajaxTransferConfig && _typeof(window.__ajaxTransferConfig) === 'object' && (
      Mmversion.isIOS && Mmversion.compareHexVersion('1800282F') || Mmversion.isAndroid && Mmversion.compareHexVersion('28002234') || Mmversion.isWindowsWechat && Mmversion.cpVersion('3.9.5', 1, true, 'windows') || Mmversion.isMacWechat && Mmversion.cpVersion('3.8.4', 1, true, 'mac') || Mmversion.isHarmonyWechat && Mmversion.compareHexVersion('0xf3100b00') && !Mmversion.compareHexVersion('0xf3100c00') || Mmversion.compareHexVersion('0xf3800b00'))) {
        try {
          return findAjaxScopeByConfig(ajaxUrl, window.__ajaxTransferConfig);
        } catch (err) {

        }
      }
    }
    function getActionByData(data) {
      var _a, _b;
      if (_typeof(data) === 'object' && !(data instanceof Blob)) {
        if (data.hasOwnProperty('data') && typeof data.data === 'string') {
          try {
            var workedData = JSON.parse(data.data);
            return workedData.action || '';
          } catch (e) {}
        }
        return data.action || '';
      }
      if (typeof data === 'string') {
        return ((_b = (_a = data.split(/[?&]/).find(function (x) {
          return x.indexOf('action=') >= 0;
        })) === null || _a === void 0 ? void 0 : _a.split('=')) === null || _b === void 0 ? void 0 : _b[1]) || '';
      }
      return '';
    }

    var METHOD_ENUM = {
      GET: 0,
      POST: 1
    };
    var __moon_report = window.__moon_report || function () {};
    var MOON_AJAX_SUCCESS_OFFSET = 3;
    var MOON_AJAX_NETWORK_OFFSET = 4;
    var MOON_AJAX_ERROR_OFFSET = 5;
    var MOON_AJAX_TIMEOUT_OFFSET = 6;
    var MOON_AJAX_COMPLETE_OFFSET = 7;
    var LENGTH_LIMIT = 4096;
    function reportRtError(type, id, key, content) {
      var log = '';
      var prefix = type === 'rt' ? 'rtCheckError' : 'Ajax Length Limit';
      if (content === null || content === void 0 ? void 0 : content.length) {
        var loglen = 1000;
        var len = content.length;
        var lc = Math.ceil(len / loglen);
        log = ["&lc=".concat(lc)];
        for (var i = 0; i = 200 && retryStatus  0 ? 'json' : undefined
          });
          var isTimeout = false;
          handleReqTimeout({
            abort: function abort() {
              isTimeout = true;
              reqLogItem.endTime = Date.now();
              reqLogItem.response = 'timeout';
              networkEndLog(reqLogItem);
            }
          });
          Device.os.pc && monitor$1.setSum(115849, 69, 1);
          JSAPI.invoke(Device.os.pc ? 'H5ExtTransfer' : 'webTransfer', params, function (res) {
            var _a, _b, _c, _d, _e, _f;
            if (isTimeout) return;
            var status = 400;
            var result = '';
            if (Device.os.pc) {
              try {
                var retFlag = res.base_resp.ret === 0 && res.jsapi_resp.ret === 0 && res.err_msg.indexOf(':ok') > -1;
                var respJsonFlag = res.jsapi_resp.resp_json;
                status = retFlag && respJsonFlag ? 200 : 400;
                result = res.jsapi_resp.resp_json;
              } catch (err) {
                console.error(err);
              }
            } else {
              status = res && res.errCode * 1 === 0 && typeof res.result === 'string' && res.result ? 200 : 400;
              result = res.result;
            }
            try {
              Log.log("ajax transfer, status: ".concat(status, ", reqUrl: ").concat(reqUrl));
            } catch (err) {
              console.error(err);
            }
            if (status >= 200 && status  -1 && retryStatus >= 200 && retryStatus = 200 && status  LENGTH_LIMIT) {
              reportAjaxLength(27613, 17, "ajax get limit[length: ".concat(url.length, "]").concat(url.substring(0, 1024)));
            }
            if (data && !(data instanceof Blob) && data.length > LENGTH_LIMIT) {
              reportAjaxLength(27613, 18, "ajax post limit[length: ".concat(data.length, "]").concat(data.substring(0, 1024)));
            }
            if (data && data instanceof Blob && data.size > LENGTH_LIMIT) {
              reportAjaxLength(27613, 18, "ajax post limit[length: ".concat(data.size, "]blob"));
            }
          } catch (e) {
          }
        } catch (e) {
          obj.error && obj.error(xhr, {
            type: 3,
            error: e,
            status: 0
          });
        }
        beforeReq();
      });
    }

    Mmversion.isWindowsWechat && Mmversion.compareHexVersion('0xf2550000') || Mmversion.isMacWechat && Mmversion.compareHexVersion('0xf2650000');

    var getBrandServiceType = function getBrandServiceType() {
      var serviceType = arguments.length > 0 && arguments[0] !== undefined ? arguments[0] : window.service_type;
      var _a, _b;
      var brandServiceType = 0;
      if (serviceType !== undefined) brandServiceType = serviceType + 1;
      if (((_b = (_a = window.cgiData) === null || _a === void 0 ? void 0 : _a.trans_appmsg_info) === null || _b === void 0 ? void 0 : _b.trans_type) * 1 === 1) brandServiceType = 3;
      return brandServiceType;
    };

    function setCurrentMpInfo(ifShow) {
      var supportNewTopBar = Mmversion.isIOS && Mmversion.gtVersion('7.0.10', true) || Mmversion.isAndroid && Mmversion.gtVersion('7.0.12', true);
      var supportLiveStatus = Mmversion.isIOS && Mmversion.gtVersion('8.0.46', true) || Mmversion.isAndroid && Mmversion.gtVersion('8.0.46', true);
      JSAPI.invoke('currentMpInfo', {
        userName: window.user_name,
        brandName: !!supportNewTopBar && window.nickname === '' ? '未命名账号' : window.title,
        title: window.msg_title || '',
        brandIcon: window.hd_head_img.replace(/\/0$/, '/132'),
        itemShowType: window.item_show_type,
        isPaySubscribe: window.isPaySubscribe,
        topBarStyle: supportNewTopBar ? 1 : 0,
        topBarShowed: ifShow,
        disableShowFinderLiveTopBar: !ifShow && supportLiveStatus ? 1 : 0,
        brandServiceType: getBrandServiceType()
      }, function () {});
    }
    function AjaxWx(obj) {
      var report36408 = typeof obj.report36408 === 'function' ? obj.report36408 : function () {};
      obj.url += obj.url.indexOf('?') === -1 ? '?fasttmplajax=1' : '&fasttmplajax=1';
      if (getAjaxScope(obj.url)) {
        Ajax(obj);
        return;
      }
      if (obj.usePb) {
        obj.type = 'POST';
        obj.data = {
          data: JSON.stringify(obj.data)
        };
      }
      if (!/^(http:\/\/|https:\/\/|\/\/)/.test(obj.url)) {
        obj.url = "https://mp.weixin.qq.com/".concat(obj.url.replace(/^\//, ''));
      } else if (/^\/\//.test(obj.url)) {
        obj.url = "https:".concat(obj.url);
      }
      if (obj.f !== 'html' && (obj.url.indexOf('?f=json') === -1 || obj.url.indexOf('&f=json') === -1)) {
        obj.url += '&f=json';
      }
      if (!obj.notJoinUrl && obj.f !== 'html') {
        obj.url = Url.joinUrl(obj.url);
      }
      Url.joinUserArticleRole(obj.url, !!obj.notJoinUrl, function (url) {
        obj.url = url;
        var urlObj = new URL(url, location.origin);
        var data = null;
        if (_typeof(obj.data) === 'object') {
          var d = obj.data;
          var ds = [];
          for (var k in d) {
            if (d.hasOwnProperty(k)) {
              ds.push("".concat(k, "=").concat(encodeURIComponent(d[k])));
            }
          }
          data = ds.join('&');
        } else {
          data = typeof obj.data === 'string' ? obj.data : null;
        }
        var header = {
          Cookie: document.cookie,
          referer: location.href
        };
        if (obj.contentType) {
          header['Content-Type'] = obj.contentType;
        } else if ((obj.type || 'GET').toUpperCase() === 'POST') {
          header['Content-Type'] = 'application/x-www-form-urlencoded; charset=UTF-8';
        }
        var reqLogItem = networkStartLog({
          method: obj.type || 'GET',
          url: obj.url,
          postData: obj.data || {},
          requestHeader: header,
          requestType: 'jsapi',
          startTime: Date.now()
        });
        var retryTime = 1;
        var jsapiRequest = function jsapiRequest(obj, data) {
          return JSAPI.invoke('request', {
            url: obj.url,
            method: obj.type,
            data: data,
            header: header
          }, function (res) {
            var _a, _b, _c, _d, _e, _f;
            if (res.err_msg.indexOf(':ok') > -1 && (!res.statusCode || res.statusCode >= 200 && res.statusCode  '27000600')) {
                if (!obj.dontReport) {
                  report36408({
                    CgiPath: urlObj.pathname || '',
                    Action: urlObj.searchParams.get('action') || getActionByData(obj.data) || '',
                    Query: urlObj.search || '',
                    PostData: obj.type === 'POST' && !(data instanceof Blob) ? data : '',
                    Method: obj.type || '',
                    RequestType: 20,
                    RetType: 1,
                    HttpCode: res.statusCode || 0,
                    Ret: ((_c = resData === null || resData === void 0 ? void 0 : resData.base_resp) === null || _c === void 0 ? void 0 : _c.ret) || 0
                  });
                }
                var _retryTime = retryTime++;
                JSAPI.invoke('updatePageAuth', {}, function (res) {
                  console.log('[skeleton] updatePageAuth', res);
                  monitor$1.setSum(112287, 3, 1);
                  if (res && res.err_msg && res.err_msg.indexOf(':ok') > -1) {
                    window.top.pass_ticket = encodeURIComponent(Url.getQuery('pass_ticket', res.fullUrl).html(false).replace(/\s/g, '+'));
                    if (obj.pass_ticket) {
                      obj.pass_ticket = window.top.pass_ticket;
                    }
                    console.warn('[skeleton] updatePageAuth resetTopbar');
                    var supportNewTopBar = Mmversion.isIOS && Mmversion.gtVersion('7.0.10', true);
                    var showBottomBar = !!window.is_login;
                    if (window.top.item_show_type === '0' && supportNewTopBar) {
                      var top = document.documentElement.scrollTop || window.pageYOffset || document.body.scrollTop || 0;
                      setCurrentMpInfo(top > 40 && !showBottomBar);
                    }
                    try {
                      obj.url = Url.addParam(obj.url, 'retry', _retryTime, true);
                    } catch (err) {
                      console.error(err);
                    }
                    jsapiRequest(obj, data);
                    monitor$1.setSum(112287, 4, 1);
                  } else {
                    obj.success && obj.success(resData);
                    obj.complete && obj.complete();
                    if (Mmversion.isIOS) {
                      monitor$1.setSum(112287, 35, 1);
                    } else {
                      monitor$1.setSum(112287, 36, 1);
                    }
                    reqLogItem.status = 200;
                    reqLogItem.endTime = Date.now();
                    reqLogItem.response = resData;
                    networkEndLog(reqLogItem);
                  }
                });
              } else {
                if (((_d = tmpResData === null || tmpResData === void 0 ? void 0 : tmpResData.base_resp) === null || _d === void 0 ? void 0 : _d.ret) !== 0) {
                  if (!obj.dontReport) {
                    report36408({
                      CgiPath: urlObj.pathname || '',
                      Action: urlObj.searchParams.get('action') || getActionByData(obj.data) || '',
                      Query: urlObj.search || '',
                      PostData: obj.type === 'POST' && !(data instanceof Blob) ? data : '',
                      Method: obj.type || '',
                      RequestType: 20,
                      RetType: 4,
                      HttpCode: res.statusCode || 0,
                      Ret: ((_e = tmpResData === null || tmpResData === void 0 ? void 0 : tmpResData.base_resp) === null || _e === void 0 ? void 0 : _e.ret) || 0
                    });
                  }
                } else {
                  if (!obj.dontReport) {
                    report36408({
                      CgiPath: urlObj.pathname || '',
                      Action: urlObj.searchParams.get('action') || getActionByData(obj.data) || '',
                      Query: urlObj.search || '',
                      PostData: obj.type === 'POST' && !(data instanceof Blob) ? data : '',
                      Method: obj.type || '',
                      RequestType: 20,
                      RetType: 0,
                      HttpCode: res.statusCode || 0,
                      Ret: ((_f = tmpResData === null || tmpResData === void 0 ? void 0 : tmpResData.base_resp) === null || _f === void 0 ? void 0 : _f.ret) || 0
                    });
                  }
                }
                obj.success && obj.success(resData);
                obj.complete && obj.complete();
                reqLogItem.status = 200;
                reqLogItem.endTime = Date.now();
                reqLogItem.response = resData;
                networkEndLog(reqLogItem);
              }
            } else if (res.err_msg.indexOf('no permission') > -1 || !Mmversion.isOnlyWechat) {
              if (!obj.dontReport) {
                report36408({
                  CgiPath: urlObj.pathname || '',
                  Action: urlObj.searchParams.get('action') || getActionByData(obj.data) || '',
                  Query: urlObj.search || '',
                  PostData: obj.type === 'POST' && !(data instanceof Blob) ? data : '',
                  Method: obj.type || '',
                  RequestType: 20,
                  RetType: 1,
                  HttpCode: res.statusCode || 0,
                  Ret: 0
                });
              }
              Ajax(obj);
              if (res.err_msg.indexOf('no permission') > -1) {
                console.warn('[JSAPI Request] No permission');
                monitor$1.setSum(112287, 31, 1);
              }
              reqLogItem.status = 302;
              reqLogItem.endTime = Date.now();
              reqLogItem.response = res;
              networkEndLog(reqLogItem);
            } else {
              if (!obj.dontReport) {
                report36408({
                  CgiPath: urlObj.pathname || '',
                  Action: urlObj.searchParams.get('action') || getActionByData(obj.data) || '',
                  Query: urlObj.search || '',
                  PostData: obj.type === 'POST' && !(data instanceof Blob) ? data : '',
                  Method: obj.type || '',
                  RequestType: 20,
                  RetType: 2,
                  HttpCode: res.statusCode || 0,
                  Ret: 0
                });
              }
              obj.error && obj.error(null, {
                type: 3,
                error: res,
                status: 0
              });
              obj.complete && obj.complete();
              monitor$1.setSum(112287, 32, 1);
              var sample = 0.001;
              if (Math.random() = 0; --o) { var i = this.tryEntries[o], a = i.completion; if ("root" === i.tryLoc) return handle("end"); if (i.tryLoc = 0; --r) { var o = this.tryEntries[r]; if (o.tryLoc = 0; --e) { var r = this.tryEntries[e]; if (r.finallyLoc === t) return this.complete(r.completion, r.afterLoc), resetTryEntry(r), y; } }, "catch": function _catch(t) { for (var e = this.tryEntries.length - 1; e >= 0; --e) { var r = this.tryEntries[e]; if (r.tryLoc === t) { var n = r.completion; if ("throw" === n.type) { var o = n.arg; resetTryEntry(r); } return o; } } throw new Error("illegal catch attempt"); }, delegateYield: function delegateYield(e, r, n) { return this.delegate = { iterator: values(e), resultName: r, nextLoc: n }, "next" === this.method && (this.arg = t), y; } }, e; }
    var AjaxRouter = function () {
      var _ref = _asyncToGenerator( _regeneratorRuntime().mark(function _callee(obj) {
        return _regeneratorRuntime().wrap(function _callee$(_context) {
          while (1) switch (_context.prev = _context.next) {
            case 0:
              if (!window.__secPageAuthPromise) {
                _context.next = 3;
                break;
              }
              _context.next = 3;
              return window.__secPageAuthPromise;
            case 3:
              if (!(!Mmversion.isWxWork && (window.__second_open__ || !getIsAcrossOrigin() && top.window.__second_open__) && window.__is_page_auth_return__ && !obj.pureHttp)) {
                _context.next = 5;
                break;
              }
              return _context.abrupt("return", AjaxWx(obj));
            case 5:
              return _context.abrupt("return", Ajax(obj));
            case 6:
            case "end":
              return _context.stop();
          }
        }, _callee);
      }));
      return function AjaxRouter(_x) {
        return _ref.apply(this, arguments);
      };
    }();

    var isx5 = navigator.userAgent.indexOf('TBS/') !== -1;
    var getDataFunc = [];
    var reportData = [];

    var specificData = {};
    function joinUrl(url) {
      var obj = {};
      if (typeof window.uin !== 'undefined') {
        obj.uin = window.uin;
      }
      if (typeof window.key !== 'undefined') {
        obj.key = window.key;
      }
      if (typeof window.pass_ticket !== 'undefined') {
        obj.pass_ticket = window.pass_ticket;
      }
      if (typeof window.wxtoken !== 'undefined') {
        obj.wxtoken = window.wxtoken;
      }
      if (typeof window.devicetype !== 'undefined') {
        obj.devicetype = window.devicetype;
      }
      if (typeof window.clientversion !== 'undefined') {
        obj.clientversion = window.clientversion;
      }
      if (typeof window.appmsg_token !== 'undefined') {
        obj.appmsg_token = window.appmsg_token;
      } else if (url.indexOf('advertisement_report') > -1) {
        new Image().src = "".concat(location.protocol, "//mp.weixin.qq.com/mp/jsmonitor?idkey=68064_13_1&r=").concat(Math.random());
      }
      obj.x5 = isx5 ? '1' : '0';
      obj.f = 'json';
      return Url.join(url, obj);
    }
    function isObj(obj) {
      return obj && _typeof(obj) === 'object';
    }
    function assign(target, source) {
      if (isObj(target) && isObj(source)) {
        for (var key in source) {
          if (Object.prototype.hasOwnProperty.call(source, key)) {
            target[key] = source[key];
          }
        }
      }
    }
    function assembleReportData(initiative) {
      var leaveReportLog = [];
      leaveReportLog.push({
        content: "[LeaveReport] specificData keys: ".concat(Object.keys(specificData)),
        timestamp: Date.now()
      });
      Log.log("[LeaveReport] specificData keys: ".concat(Object.keys(specificData)));
      console.log("[LeaveReport] specificData keys: ".concat(Object.keys(specificData)));
      var allReportData = {};
      for (var reportField in specificData) {
        if (!allReportData[reportField]) {
          allReportData[reportField] = {};
        }
        for (var i = 0; i = BATCH_SIZE) {
        batchReport();
      } else {
        if (!timeOutId) {
          timeOutId = setTimeout(function () {
            batchReport();
            clearTimeout(timeOutId);
            timeOutId = null;
          }, BATCH_TIME);
        }
      }
    }
    _leaveReport.addReport(function () {
      var repeatedReportJson = getRepeatedReportJson();
      if (!repeatedReportJson) return false;
      var reportData = [];
      for (var _i = 0, _Object$entries = Object.entries(repeatedReportJson); _i  200 || isScrolling()) {
            return;
          }
          var st = e.changedTouches[0];
          if (Math.abs(g.y - st.clientY)  5 || Math.abs(x - e.clientX) > 5)) {
            clearTimeout(timeOutEvent);
            timeOutEvent = undefined;
            typeof cancelCb === 'function' && cancelCb.call(self, e);
          }
        });
        on(el, 'mouseup', className, function () {
          mousedown = false;
          clearTimeout(timeOutEvent);
        });
        on(el, 'click', className, function () {
          if (triggerLongClick) return false;
        });
      } else {
        on(el, 'touchstart', className, function (e) {
          e.touches.length === 1 && (timeOutEvent = setTimeout(function () {
            timeOutEvent = undefined;
            cb.call(self, e);
          }, 500));
        });
        on(el, 'touchmove', className, function (e) {
          if (!timeOutEvent) return;
          var st = e.changedTouches[0];
          if (Math.abs(g.y - st.clientY) > 5 || Math.abs(g.x - st.clientX) > 5) {
            clearTimeout(timeOutEvent);
            timeOutEvent = undefined;
            typeof cancelCb === 'function' && cancelCb.call(self, e);
          }
        });
        on(el, 'touchend', className, function (e) {
          if (timeOutEvent) {
            clearTimeout(timeOutEvent);
            timeOutEvent = undefined;
          } else {
            e.preventDefault();
          }
        }, true);
      }
    }
    function doubletap(el, cb) {
      var _this = this;
      var __lastTouchVideoTs = 0;
      var realCb = function realCb(e) {
        if (Date.now() - __lastTouchVideoTs  -1;
    }
    function closest(target, className, context) {
      while (target && !matches(target, className)) {
        target = target !== context && target.nodeType !== target.DOCUMENT_NODE && target.parentNode;
      }
      return target;
    }
    function on(el, type, className, cb, flag, extra) {
      var callback;
      var handler;
      var delegator;
      if (!el) return;
      if (typeof className === 'function') {
        extra = flag;
        flag = cb;
        cb = className;
        className = '';
      }
      if (typeof className !== 'string') {
        className = '';
      }
      if (el == window && type == "load" && /complete|loaded/.test(document.readyState)) {
        return cb({
          type: "load"
        });
      }
      if (type == 'tap') return tap(el, cb, flag, className);
      if (type === 'longtap') return longtap(el, cb, flag, className, extra);
      if (type == "unload" && "onpagehide" in window) {
        type = "pagehide";
      }
      callback = function callback(e) {
        var ret = cb(e);
        if (ret === false) {
          e.stopPropagation && e.stopPropagation();
          e.preventDefault && e.preventDefault();
        }
        return ret;
      };
      if (className && className.charAt(0) == '.') delegator = function delegator(e) {
        var target = e.target || e.srcElement;
        var match = closest(target, className, el);
        if (match) {
          e.delegatedTarget = match;
          return callback(e);
        }
      };
      handler = delegator || callback;
      cb["".concat(type, "_handler")] = handler;
      if (el.addEventListener) {
        el.addEventListener(type, handler, !!flag);
        return;
      }
      if (el.attachEvent) {
        el.attachEvent("on".concat(type), handler, !!flag);
        return;
      }
    }
    function off(el, type, cb, flag) {
      if (!el) return;
      var handlerType = type;
      var handler;
      if (handlerType == 'tap') {
        if (isUseTap()) {
          handlerType = 'touchend';
          handler = cb.tap_handler && cb.tap_handler.touchend_handler ? cb.tap_handler.touchend_handler : cb;
        } else {
          handlerType = 'click';
        }
      }
      if (!handler) {
        handler = cb["".concat(handlerType, "_handler")] || cb;
      }
      if (el.removeEventListener) {
        el.removeEventListener(handlerType, handler, !!flag);
        return;
      }
      if (el.detachEvent) {
        el.detachEvent("on".concat(handlerType), handler, !!flag);
        return;
      }
      if (handlerType == 'tap' && isUseTap()) {
        if (cb.tap_handler) {
          cb.tap_handler.touchend_handler = null;
        }
        cb.tap_handler = null;
      } else {
        cb["".concat(handlerType, "_handler")] = null;
      }
    }
    function getHiddenProp() {
      if ('hidden' in document) {
        return 'hidden';
      }
      for (var i = 0; i ', '&lt;', ''];

      var replaceReverse = ['&', '&amp;', '¥', '&yen;', '', '&gt;', ' ', '&nbsp;', '"', '&quot;', '\'', '&#39;', '`', '&#96;'];
      var str = _str;
      var target;
      if (encode) {
        target = replaceReverse;
      } else {
        target = replace;
      }
      for (var i = 0; i ', '&lt;', '', '&gt;', '"', '&quot;', '\'', '&#39;', '`', '&#96;'];
      var str = _str;
      var target;
      if (encode) {
        target = replaceReverse;
      } else {
        target = replace;
      }
      for (var i = 0; i  1 && arguments[1] !== undefined ? arguments[1] : 50;
      var timeout;
      return function () {
        for (var _len = arguments.length, args = new Array(_len), _key = 0; _key  rectA.right || rectB.bottom  rectA.bottom);
    }
    var utils = {
      isNativePage: isNativePage,
      isNewNativePage: function isNewNativePage() {
        return Url.getQuery('isNativePage') === '2';
      },
      isOldNativePage: function isOldNativePage() {
        return Url.getQuery('isNativePage') === '1';
      },
      __useWcSlPlayer: false,
      isWcSlPage: function isWcSlPage() {
        return utils.__useWcSlPlayer;
      },
      getPlayerType: function getPlayerType() {
        if (isNativePage()) {
          return 2;
        }
        return 1;
      },
      getParam: function getParam(key) {
        if (!key) return null;
        var m = location.href.match(new RegExp("(\\?|&)".concat(key, "=([^&]+)")));
        return m ? m[2] : null;
      },

      insertAfter: function insertAfter(newElement, targetElement) {
        var parentElement = targetElement.parentNode;
        if (parentElement.lastChild === targetElement) {
          parentElement.appendChild(newElement);
        } else {
          parentElement.insertBefore(newElement, targetElement.nextSibling);
        }
      },
      getInnerHeight: function getInnerHeight() {
        var innerHeightFromApp = window.getInnerHeight && window.getInnerHeight();
        return innerHeightFromApp || window.innerHeight || document.documentElement.clientHeight;
      },
      getInnerWidth: function getInnerWidth() {
        return window.innerWidth || document.documentElement.clientWidth;
      },
      getScrollTop: function getScrollTop() {
        return document.documentElement.scrollTop || window.pageYOffset || document.body.scrollTop;
      },
      getDocumentHeight: function getDocumentHeight() {
        return document.body.scrollHeight;
      },
      getElementActualTop: function getElementActualTop(element) {
        var elRect = element.getBoundingClientRect();
        var actualTop = elRect.top + this.getScrollTop();
        return actualTop;
      },
      getElementTop: function getElementTop(element) {
        return element.getBoundingClientRect().top;
      },
      getElementHeight: function getElementHeight(element) {
        return element.getBoundingClientRect().height;
      },
      getOrientation: function getOrientation() {
        var _a, _b;
        return (_b = (_a = window.screen.orientation) === null || _a === void 0 ? void 0 : _a.angle) !== null && _b !== void 0 ? _b : window.orientation;
      },
      getDirection: function getDirection() {
        var orientation = arguments.length > 0 && arguments[0] !== undefined ? arguments[0] : utils.getOrientation();
        return (Mmversion.isIPad ? [90, 270] : [0, 180]).indexOf(orientation) > -1 ? 'vertical' : 'horizontal';
      },
      listenDirectionChange: function listenDirectionChange(cb) {
        var _a, _b;
        if ((_b = (_a = window.screen) === null || _a === void 0 ? void 0 : _a.orientation) === null || _b === void 0 ? void 0 : _b.addEventListener) {
          directionChangeHandlersMap[directionHandlerId] = function (e) {
            cb === null || cb === void 0 ? void 0 : cb(utils.getDirection(e.target.angle), e.target.angle);
          };
          window.screen.orientation.addEventListener('change', directionChangeHandlersMap[directionHandlerId]);
        } else {
          directionChangeHandlersMap[directionHandlerId] = function () {
            var orientation = utils.getOrientation();
            cb === null || cb === void 0 ? void 0 : cb(utils.getDirection(orientation), orientation);
          };
          window.addEventListener('orientationchange', directionChangeHandlersMap[directionHandlerId]);
        }
        return directionHandlerId++;
      },
      unlistenDirectionChange: function unlistenDirectionChange(handlerId) {
        var _a, _b;
        if ((_b = (_a = window.screen) === null || _a === void 0 ? void 0 : _a.orientation) === null || _b === void 0 ? void 0 : _b.removeEventListener) {
          window.screen.orientation.removeEventListener('change', directionChangeHandlersMap[handlerId]);
        } else {
          window.removeEventListener('orientationchange', directionChangeHandlersMap[handlerId]);
        }
        delete directionChangeHandlersMap[handlerId];
      },
      isScrollEnd: function isScrollEnd(threshold) {
        return this.getScrollTop() + this.getInnerHeight() + threshold >= this.getDocumentHeight();
      },

      listenStateChange: function listenStateChange() {
        var opt = arguments.length > 0 && arguments[0] !== undefined ? arguments[0] : {};
        stateChangeCb.push(opt.cb);
        try {
          if (parent.window.hasListenStateChange) {
            return;
          }
        } catch (error) {
        }
        JSAPI.on('activity:state_change', function (res) {
          stateChangeCb.forEach(function (callback) {
            callback(res);
          });
        });
        try {
          parent.window.hasListenStateChange = true;
        } catch (error) {
        }
      },

      listenMpPageAction: function listenMpPageAction(cb) {
        mpPageActionCb.push(cb);
        try {
          if (parent.window.hasListenMpPageAction) {
            return;
          }
        } catch (error) {
        }
        JSAPI.on('onMPPageAction', function (res) {
          mpPageActionCb.forEach(function (callback) {
            callback(res);
          });
        });
        try {
          parent.window.hasListenMpPageAction = true;
        } catch (error) {
        }
      },
      getIosMainVersion: function getIosMainVersion() {
        var versionInfo = navigator.userAgent.toLowerCase().match(/cpu iphone os (.*?) like mac os/);
        return versionInfo && versionInfo[1] && parseInt(versionInfo[1].split('_')[0], 10);
      },

      report120081: function report120081(key, times) {
        jsmonitorReport$1.setSum(120081, key, times);
        jsmonitorReport$1.send();
      },
      loadNewPageKeepingHistoryStackIfSecOpen: function loadNewPageKeepingHistoryStackIfSecOpen(url) {
        if (window.__second_open__ && typeof url === 'string' && /^https?:\/\/mp.weixin.qq.com\//.test(url)) {
          HistoryLS.set(HistoryKey, location.href, Date.now() + 10000);
        }
        location.href = "".concat(url.replace(/#.*$/, ''), "#wechat_redirect");
      },
      initNewPageHistoryStackFromSecOpen: function initNewPageHistoryStackFromSecOpen() {
        var fromUrl = HistoryLS.get(HistoryKey);
        if (fromUrl && typeof fromUrl === 'string' && /^https?:\/\/mp.weixin.qq.com\//.test(fromUrl)) {
          HistoryLS.remove(HistoryKey);
          if (history && history.replaceState && history.pushState) {
            var curUrl = location.href;
            try {
              history.replaceState({
                __mock_secopen_history_stack_reload__: 1
              }, '', fromUrl);
              history.pushState({
                __mock_secopen_history_stack_reload__: 1
              }, '', curUrl);
            } catch (e) {
              console.error('[initNewPageHistoryStackFromSecOpen]', e);
            }
          }
        }
        if (!hasListenPopstateForSecOpenReload) {
          hasListenPopstateForSecOpenReload = true;
          window.addEventListener('popstate', function (e) {
            if (e.state && e.state.__mock_secopen_history_stack_reload__ === 1) {
              location.reload();
            }
          });
        }
      },
      initWebCompt: function initWebCompt(webComptList, callback) {
        var flushCb = function flushCb() {
          while (webComptInitCb.length) {
            var cb = webComptInitCb.shift();
            cb(webComptStatus);
          }
        };
        if (Mmversion.isWechat && !Mmversion.isInMiniProgram && (Device.os.iphone && Device.os.getNumVersion() >= 10.3 && (Mmversion.gtVersion('7.0.14', 1) && Device.os.getNumVersion() = 5 || Device.os.harmony && Mmversion.compareHexVersion('0xf3800c00'))) {
          document.addEventListener('WeixinOpenTagsReady', function () {
            webComptStatus = {
              status: 'ready'
            };
            flushCb();
          });
          document.addEventListener('WeixinOpenTagsError', function (e) {
            webComptStatus = {
              status: 'error',
              error: e && e.detail && e.detail.errMsg
            };
            flushCb();
          });
          JSAPI.invoke('handleMPPageAction', {
            action: 'wxConfig',
            appid: 'wxmpfakeid',
            webComptList: webComptList,
            url: location.href
          }, function (res) {
            console.log('wx config web compt result', webComptList, res);
            Log.info('wx config web compt result', webComptList, JSON.stringify(res));
            if (res && res.err_msg && res.err_msg.indexOf(':ok') === -1) {
              webComptStatus = {
                status: 'error',
                error: res.err_msg
              };
              flushCb();
            }
            if (typeof callback === 'function') {
              callback(res);
            }
          });
        } else {
          var res = {
            err_msg: 'handleMPPageAction:fail_webcompt unsupported'
          };
          console.log('wx config web compt result', webComptList, res);
          Log.info('wx config web compt result', webComptList, JSON.stringify(res));
          webComptStatus = {
            status: 'error',
            error: res.err_msg
          };
          flushCb();
          if (typeof callback === 'function') {
            callback(res);
          }
        }
      },
      initWebComptForWcSlVideoSharePage: function initWebComptForWcSlVideoSharePage() {
        var initAfterConfWxOpen = function initAfterConfWxOpen(res) {
          if (res.err_msg.indexOf(':ok') !== -1) {
            utils.initNewPageHistoryStackFromSecOpen();
          } else {
            window.__failConfigWxOpen = true;
            Log.info('failed to config wxopen: res not ok');
            jsmonitorReport$1.setSum(221515, Device.os.iphone ? 7 : 8, 1);
            window.WX_BJ_REPORT && window.WX_BJ_REPORT.BadJs && res && window.WX_BJ_REPORT.BadJs.report('WcSlPlayer:CfgError', (window.__second_open__ ? 'secopen:' : 'h5:') + JSON.stringify(res));
          }
        };
        if (Mmversion.isAndroid) {
          var clientVer = Mmversion.getInner();
          if (clientVer > '27001037' && clientVer = '27001100') {
            utils.initWebCompt(['wxOpen' ], initAfterConfWxOpen);
          } else if (Mmversion.gtVersion('7.0.15', 1)) {
            window.__failConfigWxOpen = true;
            Log.info('failed to config wxopen: android version check failed (gt 7.0.15)');
          } else {
            window.__failConfigWxOpen = true;
            Log.info('failed to config wxopen: android version check failed');
          }
        } else if (Mmversion.isIOS) {
          if (Mmversion.gtVersion('7.0.15', 1)) {
            utils.initWebCompt(['wxOpen' ], initAfterConfWxOpen);
          } else {
            window.__failConfigWxOpen = true;
            Log.info('failed to config wxopen: ios version check failed');
          }
        } else {
          window.__failConfigWxOpen = true;
        }
      },

      getWebComptStatus: function getWebComptStatus(cb) {
        if (typeof cb !== 'function') {
          return webComptStatus;
        }
        if (webComptStatus.status === 'loading') {
          webComptInitCb.push(cb);
        } else {
          cb(webComptStatus);
        }
        return true;
      },

      supportImmersiveMode: Mmversion.isWechat && !Mmversion.isInMiniProgram && (Mmversion.isIOS && Mmversion.gtVersion('8.0.9', 1) || Mmversion.isAndroid && Mmversion.gtVersion('8.0.9', 1)),
      debounce: debounce,

      bindDebounceScrollEvent: function bindDebounceScrollEvent(fn) {
        var scrollEle = arguments.length > 1 && arguments[1] !== undefined ? arguments[1] : window;
        var wait = arguments.length > 2 && arguments[2] !== undefined ? arguments[2] : 50;
        var useCapture = arguments.length > 3 && arguments[3] !== undefined ? arguments[3] : false;
        var debounceFn = debounce(fn, wait);
        DomEvent.on(scrollEle, 'scroll', '', debounceFn, useCapture);
      },
      checkIntersect: checkIntersect,

      clickRange: function clickRange(evt) {
        var selection = window.getSelection();
        var range = selection.rangeCount && selection.getRangeAt(0);
        if (!range || range.collapsed || !range.intersectsNode(evt.target)) {
          return false;
        }
        var rangeClientRects = range.getClientRects();
        var targetLineHeight = parseFloat(getComputedStyle(evt.target).lineHeight, 10);
        var targetRect = evt.target.getBoundingClientRect();
        for (var i in rangeClientRects) {
          if (rangeClientRects.hasOwnProperty(i)) {
            var rect = rangeClientRects[i];
            var extraHeight = targetLineHeight ? (targetLineHeight - rect.height) / 2 : 0;
            if (rect.width && checkIntersect(rect, targetRect) && evt.clientX >= rect.left && evt.clientX = rect.top - extraHeight && evt.clientY  1;
    };
    var uuid = function uuid() {
      return ((1 + Math.random()) * 0x10000 | 0).toString(16).substring(1);
    };

    function _createForOfIteratorHelper(o, allowArrayLike) { var it = typeof Symbol !== "undefined" && o[Symbol.iterator] || o["@@iterator"]; if (!it) { if (Array.isArray(o) || (it = _unsupportedIterableToArray(o)) || allowArrayLike && o && typeof o.length === "number") { if (it) o = it; var i = 0; var F = function F() {}; return { s: F, n: function n() { if (i >= o.length) return { done: true }; return { done: false, value: o[i++] }; }, e: function e(_e) { throw _e; }, f: F }; } throw new TypeError("Invalid attempt to iterate non-iterable instance.\nIn order to be iterable, non-array objects must have a [Symbol.iterator]() method."); } var normalCompletion = true, didErr = false, err; return { s: function s() { it = it.call(o); }, n: function n() { var step = it.next(); normalCompletion = step.done; return step; }, e: function e(_e2) { didErr = true; err = _e2; }, f: function f() { try { if (!normalCompletion && it["return"] != null) it["return"](); } finally { if (didErr) throw err; } } }; }
    function _unsupportedIterableToArray(o, minLen) { if (!o) return; if (typeof o === "string") return _arrayLikeToArray(o, minLen); var n = Object.prototype.toString.call(o).slice(8, -1); if (n === "Object" && o.constructor) n = o.constructor.name; if (n === "Map" || n === "Set") return Array.from(o); if (n === "Arguments" || /^(?:Ui|I)nt(?:8|16|32)(?:Clamped)?Array$/.test(n)) return _arrayLikeToArray(o, minLen); }
    function _arrayLikeToArray(arr, len) { if (len == null || len > arr.length) len = arr.length; for (var i = 0, arr2 = new Array(len); i = hMin && h = sMin && s = bMin && b  0.2) {
        return true;
      }
      if ((isHsbInRange(hbsBackgroundColor, 0, 360, 0, 20, 15, 85) || isHsbInRange(hbsBackgroundColor, 0, 360, 20, 100, 15, 100)) && hbsBackgroundColor[3] > 0.2) {
        return true;
      }
      return false;
    };
    var textToSpanFn = function textToSpanFn(text, startIdx, endIdx) {
      if (!text) {
        text = '';
      }
      var span = document.createElement('span');
      if (startIdx  endIdx) return;
      var text = textNode.textContent;
      var textLen = calAccurateTextLen(text);
      if (startIdx > 0) {
        node.insertBefore(textToSpanFn(text, 0, startIdx - 1), textNode);
      }
      if (endIdx  endIdx) break;
          var childNode = node.childNodes[i];
          if (!childNode) break;
          if (childNode.nodeType === Node.ELEMENT_NODE) {
            if (filterFn && filterFn(childNode, startIdx, endIdx)) {
              var textLen = calAccurateTextLen(childNode.innerText);
              if (startIdx  1) {
              newNode = textToSpanFn(childNodeText, 0, _textLen - 1);
              node.replaceChild(newNode, childNode);
              childNode = newNode.childNodes[0];
            }
            if (startIdx >= 0 && endIdx  _textLen - 1) {
              var _match = splitTextToSpan(newNode, childNode, startIdx, _textLen - 1);
              _match && selectedNodes.push(_match);
              if (startIdx = 0xD800 && str.charCodeAt(i) = 0xD800 && str.charCodeAt(_i)  -1) {
          return true;
        }
      }
      if (opts.ignoreFlexChildren && ele.style.display === 'flex' && (ele.style.flexDirection === 'row' || ele.style.flexDirection === 'row-reverse') && ele.children.length > 1 || opts.ignoreNotWriteableChildren && (ele.getAttribute('contenteditable') === 'false' || ele.childNodes.length === 1 && ele.childNodes[0].getAttribute('contenteditable') === 'false')
      || ((_a = ele.querySelector) === null || _a === void 0 ? void 0 : _a.call(ele, ':scope > .video_iframe[data-mpvid]'))) {
        return true;
      }
      return canNotSplitEleTagName.indexOf(ele.tagName) > -1;
    }
    function isElement(node) {
      return node.nodeType === Node.ELEMENT_NODE;
    }
    function getParaListAllNodes(element, opts) {
      var childNodes = Array.from(element.childNodes);
      if (!childNodes.length) {
        return [];
      }
      var child;
      var paragraphList = [];
      for (var i = 0; i  0) {
        if (checkBadCase(keyWordsNodeList, keywordsData, 's1s_keywords')) {
          var _keyWordsNodeList$cl;
          (_keyWordsNodeList$ = keyWordsNodeList[0]) === null || _keyWordsNodeList$ === void 0 ? void 0 : (_keyWordsNodeList$.classList) === null || _keyWordsNodeList$$cl.add(BadSearchWordsClassName);
          return;
        }
        keyWordsNodeList.forEach(function (node) {
          node.keywordsData = keywordsData;
          node.classList.add(SearchWordsBoxClassName);
          node.setAttribute('link-id', "link-".concat(Date.now(), "-").concat(Math.random()));
        });
        var lastNode = keyWordsNodeList[keyWordsNodeList.length - 1];
        var i = document.createElement('i');
        i.className = SearchWordsClassName;
        lastNode.appendChild(i);
      }
      return keyWordsNodeList;
    };
    function addKeywordToHtml(html, keywordsData) {
      if (!keywordsData) return html;
      var div = document.createElement('div');
      div.innerHTML = html;
      var paraList = getParaListAllNodes(div, {
        ignorePreloadNode: true,
        ignoreNotWriteableChildren: true
      });
      try {
        wrapTextNodesWithSpan(paraList);
        var finalParaList = getParaListAllNodes(div, {
          ignorePreloadNode: true,
          ignoreNotWriteableChildren: true
        });
        keywordsData.forEach(function (data) {
          renderKeyWord(data, finalParaList);
        });
      } catch (error) {
        return html;
      }
      return div.innerHTML;
    }
    function wrapTextNodesWithSpan(paraList) {
      paraList.forEach(function (textNode) {
        if (textNode.nodeType === Node.TEXT_NODE) {
          var span = document.createElement('span');
          span.className = 'text-node-wrapper';
          span.textContent = textNode.nodeValue;
          var parent = textNode.parentNode;
          if (parent) {
            parent.replaceChild(span, textNode);
          }
        }
      });
    }

    var isAllowRender = (Mmversion.isAndroid || Mmversion.isIOS || Mmversion.isWindows || Mmversion.isMac && Mmversion.cpVersion("3.8.2", 1, true, "mac")) && Mmversion.isWechat && !Mmversion.isWxWork;
    Mmversion.isWechat && !Mmversion.isInMiniProgram && (Mmversion.isAndroid && Mmversion.compareHexVersion('28003D3C') || Mmversion.isIOS && Mmversion.compareHexVersion('18003D22'));

    var ltReplaceChar = "lt-".concat(Date.now(), "-").concat(Math.random());
    var gtReplaceChar = "gt-".concat(Date.now(), "-").concat(Math.random());
    var ltReplaceCharReg = new RegExp(ltReplaceChar, 'g');
    var gtReplaceCharReg = new RegExp(gtReplaceChar, 'g');
    function isAudioPage$1(itemShowType) {
      return itemShowType * 1 === 7;
    }
    function isImagePage(itemShowType) {
      return itemShowType * 1 === 8;
    }
    function markLink(desc) {
      var aTagRegex = /]*\bhref\s*=\s*["'][^"']*?["'][^>]*>/g;
      return desc.replace(aTagRegex, function (match) {
        return match.replace(/]*\bclass\s*=\s*["'][^"']*?\bjs_poi_entry\b[^"']*["'][^>]*>.*?)/;
        var match = desc.match(regex);
        if (match) {
          var extractedATag = match[1];
          desc = desc.replace(regex, '');
          var poiCon = document.createElement('div');
          poiCon.innerHTML = extractedATag;
          var poiNode = poiCon.getElementsByClassName('js_poi_entry')[0];
          if (poiNode) {
            var _poiNodedataset2, _poiNodedataset4, _poiNodedataset6;
            poiInfo = {
              longitude: ((_poiNode$dataset = poiNode.dataset) === null || _poiNodedataset.longitude) || '',
              latitude: ((_poiNode$dataset2 = poiNode.dataset) === null || _poiNodedataset2.latitude) || '',
              name: ((_poiNode$dataset3 = poiNode.dataset) === null || _poiNodedataset3.name) || '',
              address: ((_poiNode$dataset4 = poiNode.dataset) === null || _poiNodedataset4.address) || '',
              poiid: ((_poiNode$dataset5 = poiNode.dataset) === null || _poiNodedataset5.poiid) || '',
              content: poiNode.innerText || '',
              districtid: ((_poiNode$dataset6 = poiNode.dataset) === null || _poiNodedataset6.districtid) || ''
            };
          }
        }
      } catch (err) {
        console.error('extract old end poi fail', err);
      }
      return {
        desc: desc,
        poiInfo: poiInfo
      };
    }
    var getDesc = function getDesc(desc, isNoEncode, itemShowType) {
      var extData = arguments.length > 3 && arguments[3] !== undefined ? arguments[3] : {};
      var endPoiInfo = null;
      function getAttr(s, a) {
        var m = s.match(new RegExp(a + '\\s*=\\s*["\']?([^"\'\\s>]+)["\']?'));
        return m && m[1];
      }
      function getClassAttr(s, a) {
        var m = s.match(new RegExp(a + '\\s*=\\s*["\']?([^"\'>]+)["\']?'));
        return m && m[1];
      }
      function addFinderTags(desc, finderTransInfo) {
        var str = desc;
        if (finderTransInfo.tag_json) {
          try {
            var data = JSON.parse(finderTransInfo.tag_json);
            console.log('[finder] 插入话题标签', data);
            var topics = data.topic_info;
            topics.filter(function (topicItem) {
              return topicItem.topic_type === 1;
            }).forEach(function (topicItem) {
              var text = topicItem.topic;
              var tagHtml = ['#").concat(text, "")].join(' ');
              str = str.replace(new RegExp("#".concat(text), 'g'), tagHtml);
            });
          } catch (e) {
            console.error('[finder] 插入话题标签失败', e);
          }
        }
        if (finderTransInfo.location_json) {
          try {
            var _data = JSON.parse(finderTransInfo.location_json);
            console.log('[finder] 插入poi标签', _data);
            var text = _data.poiName;
            var poiId = _data.poiClassifyId.replace(/^qqmap_/, '');
            endPoiInfo = {
              longitude: _data.longitude,
              latitude: _data.latitude,
              name: text,
              address: _data.poiAddress,
              poiid: poiId,
              content: text,
              districtid: ''
            };
          } catch (e) {
            console.error('[finder] 插入poi标签失败', e);
          }
        }
        if (finderTransInfo.music_json && false) {
          try {
            var dataWrp = JSON.parse(finderTransInfo.music_json);
            var _data2 = dataWrp.music_info;
            console.log('[finder] 插入音频标签', _data2);
            var listenId = _data2.doc_type === 1 ? _data2.doc_id : '';
            var musicId = _data2.doc_type === 0 ? _data2.doc_id : '';
            var _text = _data2.name || '原声';
            var source = _data2.doc_type === 1 ? 0 : _data2.doc_type;
            if (listenId || musicId) {
              var tagHtml = ['").concat(_text, "")].join(' ');
              str += "\n".concat(tagHtml);
            } else {
              console.error('[finder] 插入音频标签失败 不支持的音频类型', _data2);
            }
          } catch (e) {
            console.error('[finder] 插入音频标签失败', e);
          }
        }
        console.log('[finder] 最终内容', str);
        return str;
      }
      function filterContentWithLinkNWeapp(str, enableTagReg, preserveS1SReg) {
        enableTagReg && (str = replaceTagChar(str, enableTagReg));
        var s1sMatches = [];
        if (preserveS1SReg) {
          str = str.replace(preserveS1SReg, function (match) {
            s1sMatches.push(match);
            return "__S1S_PLACEHOLDER_".concat(s1sMatches.length - 1, "__");
          });
        }
        str = str.split(/(]*>)((?:.|\n)*?)()/);
        var valid;
        for (var i = 0; i = 1) {
                r += ' class="wx_img_refer_link js_wx_tap_highlight wx_tap_link js_img_refer_link"' + ' data-seq="' + seqId + '"';
                valid = true;
              } else {
                valid = false;
              }
            } else if (weappId) {
              r += ' class="weapp_text_link"' +
              ' data-miniprogram-type="text"' + ' data-miniprogram-appid="' + weappId + '"' + ' data-miniprogram-path="' + (getAttr(str[i], 'data-miniprogram-path') || '') + '"' + ' data-miniprogram-nickname="' + (getAttr(str[i], 'data-miniprogram-nickname') || '') + '"' + ' data-miniprogram-servicetype="' + (getAttr(str[i], 'data-miniprogram-servicetype') || '0') + '"';
              valid = true;
            } else if (quoteId) {
              str[i] = "");
              str[i + 1] = '';
              str[i + 2] = "";
              i += 3;
              valid = true;
              continue;
            } else if (bizFakeId) {
              r += " class=\"js_mention_entry wx_at_link\" data-username=\"".concat(getAttr(str[i], 'data-username'), "\" data-biz=\"").concat(bizFakeId, "\" data-servicetype=\"").concat(serviceType, "\"");
              valid = true;
            } else if (plainMusicId) {
              str[i] = str[i].replace('".concat(str[i + 1], "");
              str[i + 2] = "";
              i += 3;
              continue;
            } else if (getAttr(str[i], 'tagname') === 'mp-common-product') {
              if (getAttr(str[i], 'data-cardtype') === '2') {
                str[i] = str[i].replace('".concat(str[i + 1], "");
                str[i + 2] = "";
                i += 3;
                continue;
              } else if (getAttr(str[i], 'data-cardtype') === '3') {
                var customstyle = getAttr(str[i], 'data-customstyle');
                var customHeight = void 0;
                if (customstyle) {
                  try {
                    var _JSON$parse = JSON.parse(customstyle.html(false)),
                      display = _JSON$parse.display,
                      height = _JSON$parse.height;
                    if (display !== 'none') {
                      customHeight = parseInt(height, 10);
                    } else {
                      customHeight = 0;
                    }
                  } catch (err) {
                    console.error(err);
                  }
                } else {
                  var fontScale = getScaleByDom();
                  customHeight = 24 + 44 * fontScale;
                }
                str[i] = str[i].replace('";
                i += 3;
                continue;
              }
            } else if (valid) {
              var className = getClassAttr(str[i], 'class');
              try {
                if (className.includes('normal_text_link')) {
                  var isMpWeixinLink = /^https?:\/\/mp\.weixin\.qq\.com\/s/.test(href);
                  if (isMpWeixinLink) {
                    className = className + ' mp_article_text_link';
                  }
                }
              } catch (error) {
                console.log('get class error', error);
              }
              r += " class=\"js_common_share_desc_link".concat(className ? " ".concat(className) : '', "\"");
              var _itemShowType = getAttr(str[i], 'data-itemshowtype');
              _itemShowType && (r += ' data-itemshowtype="' + _itemShowType + '"');
            }
            str[i] = valid ? r + '>' : '';
          } else if (i % 4 === 3) {
            !valid && (str[i] = '');
          } else {
            str[i] = isAudioPage$1(itemShowType) ? str[i].replace(/]+>/g, '') : str[i].replace(/]+>/g, '');
          }
        }
        str = str.join('');
        enableTagReg && (str = str.replace(ltReplaceCharReg, ''));
        if (preserveS1SReg && s1sMatches.length > 0) {
          str = str.replace(/__S1S_PLACEHOLDER_(\d+)__/g, function (_, index) {
            return s1sMatches[parseInt(index, 10)];
          });
        }
        str = str.replace(/\smp-original-line-height="[^"]*"/g, '').replace(/\smp-original-font-size="[^"]*"/g, '');
        str = str.replace(/\n/g, '');
        return str;
      }
      function modifySecondRendContent(str) {
        var blockquoteRegex = /(]*>)((?:.|\n)*?)()/g;
        str = str.replace(blockquoteRegex, function (match) {
          var quoteId = getAttr(match, 'data-quote-id');
          if (!(Array.isArray(extData.quoteList) && extData.quoteList.length > 0)) return '';
          var quoteItem = extData.quoteList.find(function (item) {
            return item.quote_id === quoteId;
          });
          if (!quoteItem) return '';
          var content = quoteItem.content;
          return "");
        });
        return str;
      }
      function getContentTimePoint(desc, validFunc) {
        function getTimePointTag(str) {
          var timeArr = str.split(':').map(function (item) {
            return parseInt(item, 10);
          });
          if (timeArr.length === 2) {
            timeArr.unshift(0);
          }
          var timeSec = timeArr[0] * 3600 + timeArr[1] * 60 + timeArr[2];
          return "").concat(str, "");
        }
        function validTimePoint(str) {
          console.log('=====[validTimePoint]', str);
          var timeArr = str.split(':').map(function (item) {
            return parseInt(item, 10);
          });
          if (timeArr.length === 1 || timeArr.length > 3 || str.includes('.')) return false;
          if (timeArr.length === 2) {
            var tempTime = timeArr.shift();
            timeArr.unshift(tempTime % 60);
            timeArr.unshift(Math.floor(tempTime / 60));
          }
          var timeSec = timeArr[0] * 3600 + timeArr[1] * 60 + timeArr[2];
          for (var i = 0; i  23) return false;
                break;
              case 1:
                if (timeArr[i]  59) return false;
                break;
              case 2:
                if (timeArr[i]  59) return false;
                break;
            }
          }
          if (window.cgiData && window.cgiData.duration && timeSec > window.cgiData.duration) return false;
          if (validFunc && validFunc instanceof Function && !validFunc(timeSec)) return false;
          return true;
        }
        var regex = new RegExp('\\b\\d{1,2}:\\d{2}(?::\\d{2})?\\b', 'g');
        return desc.replace(regex, function (match) {
          console.log('=====[match]', match);
          if (validTimePoint(match)) {
            return getTimePointTag(match);
          }
          return match;
        });
      }
      if (isNoEncode) {
        if (itemShowType * 1 === 8) {
          desc = desc.html(false);
          if (extData.send_source * 1 === 4) {
            var res = extractOldEndPoi(desc);
            if (res.desc) desc = res.desc;
            if (res.poiInfo) endPoiInfo = res.poiInfo;
          }
          if (extData.searchKeywordsData && window.cgiData) {
            window.cgiData.search_keywords = extData.searchKeywordsData;
          }
          if (window.cgiData && window.cgiData.search_keywords && isAllowRender) {
            desc = addKeywordToHtml(desc, window.cgiData.search_keywords);
          }
        }
        var searchKeywordsFilterReg;
        if (window.cgiData && window.cgiData.search_keywords && isImagePage(itemShowType)) {
          searchKeywordsFilterReg = window.cgiData.search_keywords.length > 0 ? /]*>([\s\S]*?)/g : undefined;
        }
        if (extData.transAppmsgData && extData.transAppmsgData.trans_type === 1 && extData.transAppmsgData.finder_trans_info) {
          desc = addFinderTags(desc, extData.transAppmsgData.finder_trans_info);
        }
        desc = filterContentWithLinkNWeapp(desc, isAudioPage$1(itemShowType) ? /]+?textstyle[^>]+?)>([\s\S]+?)/g : undefined, searchKeywordsFilterReg);
        desc = modifySecondRendContent(desc);
        desc = window.__emojiFormat(desc.replace(/\r/g, '').replace(/\n/g, ''));
      } else {
        desc = desc.replace(/\r/g, '').replace(/\n/g, '').replace(/\s/g, '&nbsp;');
      }
      if (itemShowType * 1 === 8) {
        var descDom = document.createElement('div');
        descDom && (descDom.innerHTML = desc);
        if (descDom) {
          descDom.childNodes.forEach(function (child, idx) {
            var lastNode;
            if (idx > 0) {
              lastNode = descDom.childNodes[idx - 1];
            }
            if (lastNode && lastNode.nodeName === 'BR' || idx === 0) {
              var isLeft = calLanguageRatio(child.textContent);
              if (isLeft) {
                if (child.nodeType === Node.TEXT_NODE) {
                  var span = document.createElement('span');
                  span.classList.add('wx_english_text_left');
                  span.textContent = child.nodeValue;
                  descDom.replaceChild(span, child);
                } else if (child.nodeType === Node.ELEMENT_NODE && child.classList && child.classList.contains('text-node-wrapper')) {
                  child.classList.add('wx_english_text_left');
                }
              }
            }
          });
          desc = descDom.innerHTML;
        }
      } else if (isAudioPage$1(itemShowType)) {
        desc = getContentTimePoint(desc);
      }
      desc = markLink(desc);
      return {
        desc: desc,
        endPoiInfo: endPoiInfo
      };
    };

    function isAudioPage(itemShowType) {
      return itemShowType * 1 === 7;
    }
    var __setDesc = function __setDesc(desc, isNoEncode, itemShowType) {
      var extData = arguments.length > 3 && arguments[3] !== undefined ? arguments[3] : {};
      desc = getDesc(desc, isNoEncode, itemShowType, extData).desc;
      if (itemShowType * 1 === 8) {
        var descDom = document.getElementById('js_image_desc');
        descDom && (descDom.innerHTML = desc);
      } else if (itemShowType * 1 === 10) {
        var dom = document.querySelector('.js_share_notice_dom');
        var _descDom = document.getElementById('js_text_desc');
        _descDom && (_descDom.innerHTML = desc);
        if (extData.superVoteId) dom && dom.classList.add('no_min_height');
        var titleDom = document.getElementById('js_text_title');
        var titleRect = titleDom ? titleDom.getBoundingClientRect() : undefined;
        var descRect = _descDom ? _descDom.getBoundingClientRect() : undefined;
        if (titleRect && titleRect.height > 17 * 1.4 + 2 || descRect && descRect.height > 17 * 1.6 + 2 || extData.superVoteId) {
          dom && dom.classList.add('text_align_left');
        }
      } else if (itemShowType * 1 === 7) {
        document.querySelector('#js_audio_desc') && (document.querySelector('#js_audio_desc').innerHTML = desc);
      } else {
        var _descDom2 = document.getElementById('js_common_share_desc');
        var descDomWrap = document.getElementById('js_common_share_desc_wrap');
        if (!_descDom2 || !descDomWrap) {
          return;
        }
        _descDom2.innerHTML = desc;
        if (itemShowType * 1 !== 5) {
          setTimeout(function () {
            var folderSwitcher = document.getElementById('js_folder_text_switch');
            if (_descDom2.offsetHeight - descDomWrap.offsetHeight > 1) {
              descDomWrap.className += ' weui-ellipsis_multi';
              folderSwitcher.style.display = 'block';
            } else {
              folderSwitcher.style.display = 'none';
            }
          }, 300);
        }
      }
    };
    if (!window.__second_open__) {
      var videoContentNoEncode = window.a_value_which_never_exists || '知识图谱能实现推理式的检索，用到这种技术后的 RAG 叫做 GraphRAG\x0a\x0a这是一个可以写到简历上的亮点\x0a\x0a这节刚写完，知识库项目的第 4 节，马上在 \x3ca data-unique-id=\x22msswpjqe-6seee7\x22 href=\x22https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzYzNzI2MTI2Nw==\x26amp;action=getalbum\x26amp;album_id=4306749160512208899\x26amp;scene=142\x26amp;sessionid=#wechat_redirect\x22 class=\x22normal_text_link album\x22 target=\x22_blank\x22 data-itemshowtype=\x22\x22\x3eAgent 全栈开发通关秘籍\x3c/a\x3e 更新\x0a';
      var TextContentNoEncode = window.a_value_which_never_exists || '';
      var ContentNoEncode = window.a_value_which_never_exists || '知识图谱能实现推理式的检索，用到这种技术后的 RAG 叫做 GraphRAG\x0a\x0a这是一个可以写到简历上的亮点\x0a\x0a这节刚写完，知识库项目的第 4 节，马上在 \x3ca data-unique-id=\x22msswpjqe-6seee7\x22 href=\x22https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzYzNzI2MTI2Nw==\x26amp;action=getalbum\x26amp;album_id=4306749160512208899\x26amp;scene=142\x26amp;sessionid=#wechat_redirect\x22 class=\x22normal_text_link album\x22 target=\x22_blank\x22 data-itemshowtype=\x22\x22\x3eAgent 全栈开发通关秘籍\x3c/a\x3e 更新\x0a';
      var itemShowType = window.a_value_which_never_exists || '5';
      var content = window.a_value_which_never_exists || '';
      var desc = window.a_value_which_never_exists || '知识图谱能实现推理式的检索，用到这种技术后的 RAG 叫做 GraphRAG\x0a\x0a这是一个可以写到简历上的亮点\x0a\x0a这节刚写完，知识库项目的第 4 节，马上在 \x26lt;a data-unique-id=\x26quot;msswpjqe-6seee7\x26quot; href=\x26quot;https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzYzNzI2MTI2Nw==\x26amp;amp;action=getalbum\x26amp;amp;album_id=4306749160512208899\x26amp;amp;scene=142\x26amp;amp;sessionid=#wechat_redirect\x26quot; class=\x26quot;normal_text_link album\x26quot; target=\x26quot;_blank\x26quot; data-itemshowtype=\x26quot;\x26quot;\x26gt;Agent 全栈开发通关秘籍\x26lt;/a\x26gt; 更新\x0a';
      var superVoteId = window.a_value_which_never_exists || '';
      var quoteList = null;
      try {
        quoteList = JSON.parse('[]');
      } catch (e) {
        console.error(e);
      }
      var imgList = null;
      try {
        var imgListStr = '[]';
        imgList = JSON.parse(imgListStr);
      } catch (e) {
        console.error(e);
      }
      var extData = {
        superVoteId: superVoteId,
        quoteList: quoteList,
        isFinderImage: false,
        transAppmsgData: +itemShowType === 8 ? window.cgiData && window.cgiData.trans_appmsg_info : null,
        send_source: window.a_value_which_never_exists || '' * 1,
        imgList: imgList
      };
      if (videoContentNoEncode) {
        __setDesc(videoContentNoEncode, true, itemShowType, extData);
      } else if (itemShowType * 1 === 10 && (ContentNoEncode || superVoteId) || isAudioPage(itemShowType) && ContentNoEncode) {
        __setDesc(ContentNoEncode, true, itemShowType, extData);
      } else if (TextContentNoEncode) {
        __setDesc(TextContentNoEncode, true, itemShowType, extData);
      } else if (itemShowType * 1 === 8) {
        __setDesc(content || desc, true, itemShowType, extData);
      } else {
        __setDesc(desc, false, itemShowType, extData);
      }
      window.__setDesc = __setDesc;
    }

    return __setDesc;

})();

---

**公众号：** 神光的幸福生活 | **发布时间：** 2026-08-14 20:12:54 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247487136&idx=1&sn=87181122c4278990846f13536db15537&chksm=f10bdf08f4f539ffb896d33fedcf07c838f3db6a0ffcf5d6f8b85f67ea8cb9eaa55f1c89b30c&scene=27#wechat_redirect
