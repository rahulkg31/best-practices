# Best Practices in Java



## Table of Contents

1. [Naming conventions](#Naming)

2. [Indentation and formatting]()

3. [Comments]()

4. [Logging]()

5. [Exception handling]()

6. [Class structure]()

7. [Constants and configurations]()

8. [Unit tests]()

9. [Miscellaneous]()

   


## Naming conventions

- Class names should be in CamelCase and start with an uppercase letter (e.g., `MyClass`).
- Method and variable names should be in camelCase and start with a lowercase letter (e.g., `myMethod`, `myVariable`).
- Constant names should be in uppercase with underscores separating words (e.g., `MY_CONSTANT`).
- Package names should be in lowercase and follow a reversed domain name convention (e.g., `com.example.myapp`).
- Use hyphens with all lowercase letters for project name  (e.g., `my-awesome-project`).
- Use meaningful and descriptive names to enhance code understanding (e.g., `online-shopping-system`        , `ShoppingCart`, `items`, `addItemToCart(Item item)`, `processOrder()`). 



## Indentation and formatting

- Use consistent indentation using either tabs or spaces (typically 4 spaces).
- Place opening braces `{` on the same line as the statement or declaration, and closing braces `}` on a new line.

  - ```java
    if (condition) {
        // Code block
    } else {
        // Code block
    }
    ```
- Keep line lengths within a reasonable limit (typically 80-120 characters) for better readability.
- Use blank lines to separate logical sections of code, such as between methods, class members, or different blocks of code. 
- Use proper spacing around operators, parentheses, commas,  method invocations, parameter lists, and control statements for readability-

  - ```java
    public class Example {
        public static void main(String[] args) {
            // Method invocation
            myMethod(arg1, arg2, arg3);
    
            // Array initialization
            int[] numbers = { 1, 2, 3, 4, 5 };
    
            // Parentheses and operators
            int result = (a + b) * (c - d);
    
            // Variable declarations
            int x = 10, y = 20, z = 30;
            String name1 = "John", name2 = "Jane", name3 = "Alice";
        }
    
        public static void myMethod(int a, int b, int c) {
            // Method implementation
        }
    }
    ```
  
- Align related code to improve readability and maintain a consistent  structure. For example, aligning assignment statements or variable  declarations-

  - ```java
    int age      = 25;
    String name  = "John";
    double salary = 5000.0;
    ```

    

## Comments

- Use comments to explain complex logic, provide API documentation, and add clarification where necessary.

- **Javadoc comments:**

  - Use Javadoc comments (`/** ... */`) to document classes, methods, and variables that are part of the public or protected API of your code.

  - Javadoc comments are specifically used for generating API documentation, so they should provide detailed information about the usage, parameters, return values, exceptions, and any relevant notes or examples.

  - ```java
    /**
     * This is a Javadoc comment for the MyClass class.
     * It provides detailed documentation about the class.
     */
    public class MyClass {
        /**
         * This is a Javadoc comment for the myMethod() method.
         * It provides detailed documentation about the method.
         *
         * @param x the first parameter
         * @param y the second parameter
         * @return the sum of x and y
         * @throws IllegalArgumentException if x or y is negative
         */
        public int myMethod(int x, int y) throws IllegalArgumentException {
            // Method implementation goes here
        }
    }
    ```

- **Normal comments:**

  - Use normal comments (**inline**`// ...` or **multiline plus inline** `/* ... */`) for providing explanations, clarifications, or context within your code.

  - ```java
    public class Example {
        public static void main(String[] args) {
            int x = 10; // Initialize variable x with a value of 10
            int y = 20; // Initialize variable y with a value of 20
    
            /*
             * Calculate the sum of x and y
             * and store it in the result variable.
             */
            int result = x + y;
    
            /* Print the result to the console.*/
            System.out.println(result);
        }
    }
    ```



## Logging

Logging conventions in Java help ensure consistency and readability in  log messages across your codebase. Here are some common logging conventions:

- Use meaningful log messages - log messages should provide relevant information about the event or condition being logged.

-  Log statements should include an appropriate log level to indicate the severity of the message. Common log levels include `DEBUG`, `INFO`, `WARN`, `ERROR`, and `FATAL`.

- Use placeholders for dynamic data instead of concatenating the values directly into the log message. This allows for better performance and readability. For example, use `{}` as a placeholder and provide the actual value using parameterized logging.

  - ```java
    logger.info("Processing request for user '{}'", username);
    logger.debug("Processing time for task '{}' is {} milliseconds", taskName, duration);
    ```

- Avoid logging sensitive information.

  - ```java
    // Avoid logging passwords or sensitive data
    logger.info("User '{}' logged in", username);
    ```



## Exception handling

Exception handling is an important aspect of Java programming, and  following conventions can help ensure consistency and maintainability in your code.

- Use specific exception types instead of catching the general `Exception` type.

  - ```java
    try {
        // Code that may throw a specific exception
    } catch (IOException e) {
        // Exception handling specific to IOException
    } catch (SQLException e) {
        // Exception handling specific to SQLException
    }
    ```

- Provide meaningful error messages and log them.

  - ```java
    try {
        // Code that may throw an exception
    } catch (IOException e) {
        logger.error("Error occurred while reading the file: " + e.getMessage(), e);
    }
    ```

- Use `finally` blocks to clean up resources or perform  necessary cleanup actions, regardless of whether an exception occurs or  not. This ensures proper resource management and cleanup in scenarios  where exceptions are thrown.

  - ```java
    Connection connection = null;
    try {
        // Code that establishes a database connection
        connection = DriverManager.getConnection(url, username, password);
        // Code that uses the database connection
    } catch (SQLException e) {
        // Exception handling specific to database errors
    } finally {
        if (connection != null) {
            try {
                connection.close();
            } catch (SQLException e) {
                // Exception handling specific to connection closing errors
            }
        }
    }
    ```

- Use try-with-resources when working with resources that implement the `AutoCloseable` interface (such as streams or database connections), use the try-with-resources statement to automatically close the resources, even if an exception occurs.

  - ```java
    try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
        // Code that uses the BufferedReader
    } catch (IOException e) {
        // Exception handling specific to IO errors
    }
    ```

    

## Class structure

- Organize classes in logical packages based on their functionality and purpose.

  - ```css
    com.example.cardealership
    ├── model
    │   ├── Car.java
    │   ├── Customer.java
    │   └── Sale.java
    ├── service
    │   ├── CarService.java
    │   ├── CustomerService.java
    │   └── SaleService.java
    ├── repository
    │   ├── CarRepository.java
    │   ├── CustomerRepository.java
    │   └── SaleRepository.java
    └── Main.java
    ```

- Place import statements at the beginning of the file and use specific imports instead of wildcards (`import java.util.List;` instead of `import java.util.*;`).

- Follow a standard order of class members: static fields, instance fields, constructors, methods.

  - ```java
    public class MyClass {
        
        // Static fields
        private static int staticField1;
        private static String staticField2;
        
        // Instance fields
        private int instanceField1;
        private String instanceField2;
        
        // Constructors
        public MyClass() {
            // Default constructor
        }
        
        public MyClass(int instanceField1, String instanceField2) {
            this.instanceField1 = instanceField1;
            this.instanceField2 = instanceField2;
        }
        
        // Methods
        public static void staticMethod() {
            // Static method implementation
        }
        
        public void instanceMethod() {
            // Instance method implementation
        }
        
        // Getters and setters
        // ...
        
        // Other helper methods
        // ...
    }
    ```



## Constants and configurations

Constants provide maintainability, readability, and reusability, while  configuration files offer flexibility, security, and  environment-specific settings.

- Constants are typically defined using the `final` modifier and uppercase letters with underscores for word separation.

  - ```java
    public class MathUtils {
        public static final double PI = 3.14159;
        public static final int MAX_ATTEMPTS = 5;
    }
    ```

- For configuration values, it's common to use properties files or configuration classes. In the following example, the configuration values are stored in an `application.properties` file. They are by defult put in `src/main/resources`.

  - ```properties
    # application.properties
    
    # Database configuration
    db.host=localhost
    db.port=3306
    db.username=myuser
    db.password=mypassword
    
    # Email configuration
    email.smtp.host=smtp.example.com
    email.smtp.port=587
    email.smtp.username=myemail@example.com
    email.smtp.password=mypassword
    ```



## Unit tests

Unit testing is an essential practice in software development to ensure  the correctness and reliability of individual units of code.

- **Test One Behaviour Per Test:** Each unit test should focus on testing a single behaviour or  functionality. This helps maintain test clarity and allows for easier  troubleshooting.

- **Use Descriptive Test Method Names:** Choose clear and descriptive names for your test methods that convey the behavior being tested. This enhances readability and understanding of  the test cases.

  - ```java
    @Test
    public void addItemShouldIncreaseItemCount() {
        // Test logic
    }
    
    @Test
    public void addItemShouldNotAddDuplicateItems() {
        // Test logic
    }
    ```

- **Arrange-Act-Assert (AAA) Pattern:** Follow the AAA pattern in your test methods, where you arrange the test  environment, perform the necessary actions, and then assert the expected outcomes. 

  - ```java
    @Test
    public void calculateDiscountShouldApplyPercentageDiscountWhenTotalExceedsThreshold() {
        // Arrange
        ShoppingCart cart = new ShoppingCart();
        cart.addItem(new Item("Item 1", 100.0));
        cart.addItem(new Item("Item 2", 200.0));
    
        // Act
        double discount = cart.calculateDiscount();
    
        // Assert
        assertEquals(20.0, discount, 0.01);
    }
    ```

- **Test Edge Cases and Boundaries:** Test scenarios at the edges and boundaries of input values to ensure the code handles them correctly. This helps identify potential issues or  unexpected behavior.

- **Mock External Dependencies:** Use mocking frameworks (e.g., Mockito) to isolate the unit under test  from its dependencies. This allows for focused testing of the unit's  behavior. 

  - ```java
    import static org.mockito.Mockito.*;
    
    @RunWith(MockitoJUnitRunner.class)
    public class UserServiceTest {
    
        @Mock
        private UserRepository userRepository;
    
        @InjectMocks
        private UserService userService;
    
        @Test
        public void getUserByIdShouldReturnUserWhenUserExists() {
            // Arrange
            long userId = 123;
            User expectedUser = new User(userId, "John Doe");
            when(userRepository.getUserById(userId)).thenReturn(expectedUser);
    
            // Act
            User actualUser = userService.getUserById(userId);
    
            // Assert
            assertEquals(expectedUser, actualUser);
            verify(userRepository, times(1)).getUserById(userId);
        }
    }
    
    ```

- **Test Driven Development (TDD):** Consider adopting a TDD approach, where tests are written before the  implementation. This helps in designing cleaner and more modular code. 



## Miscellaneous

- Use of final keyword
  - Mark constants with the `final` keyword (`private static final int MAX_COUNT = 10;`).
  - Use `final` for variables or method parameters that should not be reassigned.

- Avoid unnecessary or excessive comments, exceptions, empty code blocks, or unused code.
- **Avoid code duplication:** DRY (Don't Repeat Yourself) principle. Extract reusable code into separate methods or classes to avoid duplication and improve maintainability.
- **Optimize performance:** Write efficient code by considering performance implications. Use appropriate data structures and algorithms, minimize unnecessary object creations, optimize loops, and be mindful of I/O operations.
- **Document your code:** Provide clear and concise documentation for your code, including method and class-level comments, explaining their purpose, input/output, and any relevant usage instructions. Use tools like Javadoc to generate API documentation.
- **Use version control:** Utilize a version control system (e.g., Git) to track changes, collaborate with others, and manage your codebase effectively. Follow best practices for branching, merging, and committing code.
- **Continuously refactor and improve:** Regularly review and refactor your code to improve its design, performance, and maintainability. Apply design patterns, remove code smells, and address technical debt.

