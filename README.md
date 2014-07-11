Console-Provider
================
/*
 console-provider.js
 ===================
 Single responsibility: provide the caller with a console which they can use to shadow the global console object.
 Based initially on https://github.com/paulmillr/console-polyfill

 Usage:
 ------
 * as you're developing, just use console.log as per normal
 * when you've finished developing, if your console.log's are TOO DAMN GOOD TO REMOVE, just require-in 'console-provider' to consoleProvider, then add this line at the top of your file:

 define(['console-provider', ...],
	function(consoleProvider, ...) {
 ...
 var console = consoleProvider.getConsoleFor('yourFileName.js')

 ... and the global console object will be shadowed with an appropriate one, depending on the configuration for 'yourFileName.js' towards the bottom of this file.
 The default behaviour is to use the standard console on local, and null console on dev/staging/prod, but in future we could have e.g. in-memory logs, which could be passed back out to integration tests

 If the console statement is in a performance-sensitive part of the app (e.g. a tight loop), you can do the following to avoid even the overhead of evaluating console.log() arguments:

 !console.off && console.log('whatever');

 ... since console.off is only set on the null logger. Note that this syntax is compatible with the standard console object if you ever take console-provider out in the future.

 if you want to make use of the different logging levels (trace, debug, log, warn, error) you can also do:

 !console.traceOff && console.trace('whatever')

 .. and then suppress those by configurng the console for that unit to be:

 			'forecast-model.js': standardConsole.downToLevel('log')
*/
