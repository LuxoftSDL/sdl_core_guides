# SDL Logger

Open source version of SDL is using log4cxx for logging.
Because of lack of log4cxx stability OEM have an ability to replace log4cxx logger with any other logging boost or syslog.


SDL implements Logger abstration for easily replace library for logging.

|||
High Level Design
![TM](./assets/high_level_design.png)
|||


All components have aceess to `Logger` interface and use it for logging.

## SDLLogger

`Logger` interface is implemented by `SDLLoggerImpl`. 

SDLLoggerImpl use message loop thread to proxy loggin message to thard party (external) logger.
SDLLoggerImpl owns `ExternalLogger` and controlls it's life cycle. 


### Message loop thread in SDLLogger

Message loop thread is needed to avoid significant performance degradation in run time is logging calls are blocking calls and takes to much time. 
`SDLLoggerImpl::PushLog` is non blocking call. It will put log message in to the queue and exit immediately 


If `ExternalLogger` supports non blocking threaded logging, minor changes in SDLLogger are required : `SDLLoggerImpl::PushLog` should be reimplemented to 
use `ExternalLogger::ForceLog()` directly. 

## Logger singleton 

Logger is the only one singleton class in SDL.
Singleton pattern required to have an access to logger from any component. 
Logger singleton provides singleton by `Logger` interface. 
So sdl components do not have information about neither logger implementation nor specific external logger. 

## Logger singleton with plugins 

SDL plugins are shared librariess, so LoggerSingleton could not be implemented with Mayers singleton. 
Mayers singleton would create own SDL logger instance.

The idea is to pass singleton pointer to Plugin during creation, so that plugin could initialize Logger::instance pointer with received from sdl_core. 


Instance implementation : 
```cpp
static Logger& instance(Logger* pre_init = nullptr);

Logger& Logger::instance(Logger* pre_init) {
  static Logger* instance_ = nullptr;
  if (pre_init) {
    assert(instance_ == nullptr);
    instance_ = pre_init;
  }
  assert(instance_);
  return *instance_;
}
```

`pre_init` is `nullptr` by default, so all components will access instance_ static pointer for logging. 
`main()`  function neet to create SDLLogger implementation and call Logger::instance(logger_implementation);

Plugin implementation:
```cpp 
extern "C" PluginType* Create(Logger* logger_singleton_instance) {
  Logger::instance(logger_instance);
  return new PluginType();
}
```

SDL Core part will give pointer to logger singleton to the plugin so that plugin shared lib could initialize `Logger::instance` with the same pointer as core part. 

## Logger detailed design :


Logger int
