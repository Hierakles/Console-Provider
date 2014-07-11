define([],
	function() {

		var isIpad = navigator.userAgent.match(/iPad/i) != null;
		var isIPhone = (navigator.userAgent.match(/iPhone/i) != null) || (navigator.userAgent.match(/iPod/i) != null);
		if (isIpad) {
		  isIPhone = false;
		}

		// modify the argument to ensure it has functions which match the Chrome console API
		var polyfillConsole = function(consoleToPolyfill) {
			var prop, method;
			var empty = {};
			var dummy = function() {};
			var properties = 'memory'.split(',');
			var methods = ('assert,count,debug,dir,dirxml,error,exception,group,' +
				'groupCollapsed,groupEnd,info,log,markTimeline,profile,profileEnd,' +
				'time,timeEnd,trace,warn').split(',');
			while (prop = properties.pop()) consoleToPolyfill[prop] = consoleToPolyfill[prop] || empty;
			while (method = methods.pop()) consoleToPolyfill[method] = consoleToPolyfill[method] || dummy;

			// patch the group methods on iPad to point to log instead, since they don't work properly in safari debug tools from a mac
			if (isIpad || isIPhone ) {
				consoleToPolyfill.group = consoleToPolyfill.log;
				consoleToPolyfill.groupCollapsed = consoleToPolyfill.log;
				consoleToPolyfill.groupEnd = dummy;
			}

			return consoleToPolyfill;

		};

		// return a new object which only calls the underlying console for events at a minimum level
		var levelCheckingConsole = function(underlyingConsole, minimumLevel) {
			var levels = 'trace debug log warn error';
			if (levels.indexOf(minimumLevel) < 0) throw new Error("don't understand minimumLevel: " + minimumLevel);
			var method;
			var wrapper = {};
			var dummy = function() {};
			var methods = ('assert,count,debug,dir,dirxml,error,exception,group,' +
				'groupCollapsed,groupEnd,info,log,markTimeline,profile,profileEnd,' +
				'time,timeEnd,trace,warn').split(',');
			while (method = methods.pop()) {
				if (underlyingConsole[method]) {
					wrapper[method] = underlyingConsole[method].bind(underlyingConsole)
				} else {
					wrapper[method] = dummy;
				}

			}

			// redefine the levelled functions, to either call the underlying or not
			var levelledMethods = ('trace,debug,log,warn,error').split(',');
			while (method = levelledMethods.pop()) {
				if (levels.indexOf(method) >= levels.indexOf(minimumLevel)) {
					wrapper[method] = underlyingConsole[method].bind(underlyingConsole); // bind shenanigans for chrome: http://stackoverflow.com/questions/8159233/typeerror-illegal-invocation-on-console-log-apply
				} else {
					wrapper[method] = dummy;
				}
			}

			wrapper.traceOff = (levels.indexOf('trace') < levels.indexOf(minimumLevel));
			wrapper.debugOff = (levels.indexOf('debug') < levels.indexOf(minimumLevel));
			wrapper.logOff = (levels.indexOf('log') < levels.indexOf(minimumLevel));
			wrapper.warnOff = (levels.indexOf('warn') < levels.indexOf(minimumLevel));
			wrapper.errorOff = false;

			return wrapper;
		};

		var standardConsole = polyfillConsole(console || {});
		standardConsole.downToLevel = function(downToLevel) {
			return levelCheckingConsole(standardConsole, downToLevel);
		};

		var nullConsole = polyfillConsole({});
		nullConsole.off = true;
		nullConsole.traceOff = true;
		nullConsole.debugOff = true;
		nullConsole.logOff = true;
		nullConsole.warnOff = true;
		nullConsole.errorOff = true;
		var consoleProvider = {};

		//	http://stackoverflow.com/a/10996421
		var createRingBuffer = function(length){
			var pointer = 0, buffer = [];
			return {
				get: function(key){return buffer[key];},
				push: function(item){
					buffer[pointer] = item;
					pointer = (length + pointer +1) % length;
				},
				buffer : buffer,
				asArray: function() {
					var pointerToEnd = buffer.slice(pointer);
					var startToPointer = buffer.slice(0, pointer);
					return pointerToEnd.concat(startToPointer);
				}
			};
		};

		// recordingConsole records log's and above, to try not to impair performance too much
		var buffer = consoleProvider.buffer = createRingBuffer(500);
		var recordingConsoleFor = function(forWhom) {
			var recordCall = function(level, callArgs) {
				var message = '';
				for (var i = 0; i < callArgs.length; i++) {
					if (i > 0) message += " | ";
					var arg = callArgs[i];
					if (arg === undefined) continue;
					if (typeof arg === 'object') {
						try {
							var asJSON = JSON.stringify(callArgs[i]);
							message += asJSON;
						} catch (err){
							message += " (cannot convert to JSON -" + err.message + ")"
						}
					} else {
						message += callArgs[i].toString();
					}
				}
				var now = new Date();
				buffer.push({
					date: now,
					dateString: JSON.stringify(now), // date as a string for when it gets passed back to c# land via selenium, pure date type doesn't seem to work
					forWhom: forWhom,
					level: level, message: message});
			};

			this.log = function() {
				recordCall('log', arguments)
			};

			this.warn = function() {
				recordCall('warn', arguments)
			};

			this.error = function() {
				recordCall('error', arguments)
			};

			var timeLabels = [];
			this.time = function(label) {
				timeLabels[label] = new Date();
			};

			this.timeEnd = function(label) {
				var finished = new Date();
				var started = timeLabels[label];
				if (started === undefined) return;
				var msTaken = finished - started;
				this.log(label + " " + msTaken + "ms");
				delete timeLabels.label;
			};

			this.off = false;
			this.traceOff = true;
			this.debugOff = true;
			// record log, warn, error
			this.logOff = false;
			this.warnOff = false;
			this.errorOff = false;
			return polyfillConsole(this)
		};

		consoleProvider.getConsoleFor = function(forWhom) {
			// TODO: use the environment service to always give nullConsole unless running locally
			var defaultConsole = new recordingConsoleFor(forWhom);
//			var defaultConsole = standardConsole;
			return consoleFor[forWhom] || defaultConsole;
		};

		// uncomment a line below to use a console other than the default (null) console for a particular file
		var consoleFor = {
			'' : standardConsole
//			,'bank-feed-wizard.module.js': standardConsole
//			,'forecast-model.js': standardConsole
//			,'forecast-model.tests.js': standardConsole
//			,'label-adjuster.js': standardConsole
//			,'mojito.server.module.js': standardConsole
//			,'messagebus.module.js': standardConsole
//			,'overview.behaviour.js': standardConsole
//			,'overview.balances.monthly.viewmodel.js': standardConsole
//			,'overview.balances.yearly.viewmodel.js': standardConsole
//			,'overview.forecast.renderer.js': standardConsole
//			,'overview.forecast.viewmodel.js': standardConsole.downToLevel('trace')
//			,'overview.module.js': standardConsole
//			,'overview.renderer.js': standardConsole
//			,'overview.renderer.zoom.js': standardConsole
//			,'overview.spend-by-reason.renderer.js': standardConsole
//			,'overview.spend-by-reason.viewmodel.js': standardConsole
			//,'main.js': standardConsole
		};

		window.consoleProvider = consoleProvider;

		// if a user really wants the original without having to add it to console.for above, they can go:
//		var console = window.console.original || window.console;

		consoleProvider.original = standardConsole;
		return consoleProvider;
	}
);