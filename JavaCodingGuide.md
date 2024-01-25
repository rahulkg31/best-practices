### WHy declare logger with private, static and final?
- Example - `private static final Logger logger =  LoggerFactory.getLogger(App.class);`
- Usually, logger instances are specific to a class, no point in making these available to outside of the class. So, it is recommended to make them private.
- If the logger is defined as static, only one logger instance is created and used by all the other references of given class. Instantiation of a new Logger is a fairly expensive operation.
- Once a logger instance is defined, usually we do not change the properties of it during the application lifetime. So, it is defined with final.

### Utility classes - when to use just static methods vs singleton instance?
- Singleton patterns are used where the instance has state that may be preserved or altered across a number of calls - this might be a connection pool or some other shared object that the class provides access to.
- Static utility classes are used where each individual method is stateless and static. Utility class scope MUST be small and domain logic MUST NOT be encapsulated in static methods.
