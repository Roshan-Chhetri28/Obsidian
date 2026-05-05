## The Problem With Side Effects & Tests
1. **Testing Example**: The speaker illustrates how to write a test for a function named `writeData`, which saves data to a file using the `writeFile` method. The setup involves using the Vitest framework.
    
2. **Understanding Side Effects**: While the test can be executed successfully, it creates a file on the hard drive, which is a side effect. This raises issues such as accidentally overwriting important data or affecting production environments.
    
3. **Avoiding Side Effects**: The speaker emphasizes that tests should avoid altering the state of external systems, like databases or files, to keep tests reliable and focused on verifying code correctness without executing external functions.
    
4. **Alternative Testing Strategies**: To combat side effects, developers are encouraged to use techniques like mocking or stubbing external dependencies. This allows for testing the functionality of the code without incurring actual side effects.
    
5. **Conclusion**: The lecture underscores the importance of managing side effects in testing and promotes practices that ensure code correctness while preventing unintended consequences.

<hr>

## What is spies and mocks 
1. **Spies**: These are wrappers around functions or empty replacement functions that help track if a function was called and what arguments it received, without delving into the internal workings. Spies can be implemented by wrapping the original function or replacing it entirely, with the latter helping to avoid side effects.
    
2. **Mocks**: Mocks also involve replacing functions but typically target larger parts of an API or module. They may include test-specific logic to simulate different scenarios, allowing for more tailored testing compared to the original function.
    
3. **Importance**: Understanding spies and mocks is crucial for effective testing, especially in libraries like Vitest and Jest. The upcoming sections will showcase practical applications of these concepts.

## Details about spies and mocks
1. **Spies**:
    
    - A spy is a function that wraps another function, allowing the tester to track calls to the function and the arguments with which it was called. However, a spy does not change the original implementation of the function.
    - It serves the primary purpose of observation, meaning you can check if a function was called and with what parameters, but the actual functionality remains intact.
2. **Mocks**:
    
    - A mock is typically a more extensive replacement of a module or function with a custom implementation for testing purposes. When you mock a function, you can define specific behaviors or return values for tests.
    - This is particularly useful when the original functionality might have side effects or dependencies you wish to control. Mocks can simulate different scenarios without executing real code.