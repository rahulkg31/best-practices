### Declare logger with private, static and final.
- Example - `private static final Logger logger =  LoggerFactory.getLogger(App.class);`
- Usually, logger instances are specific to a class, no point in making these available to outside of the class. So, it is recommended to make them private.
- If the logger is defined as static, only one logger instance is created and used by all the other references of given class. Instantiation of a new Logger is a fairly expensive operation.
- Once a logger instance is defined, usually we do not change the properties of it during the application lifetime. So, it is defined with final.

### 
