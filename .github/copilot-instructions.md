# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.0
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

**Stockmarket Adapter Specifics:**
This adapter integrates stock market data into ioBroker, providing real-time and scheduled stock price information. The adapter:
- Polls stock market APIs on a scheduled basis (every 15 minutes by default)
- Fetches stock prices for user-configured symbols
- Creates ioBroker states for stock data (price, change, percentage change, volume, etc.)
- Requires an API key for external stock market data services
- Operates in schedule mode with configurable intervals
- Focuses on reliable data collection and error handling for network/API failures

## Testing

### Unit Testing
- Use Jest as the primary testing framework for ioBroker adapters
- Create tests for all adapter main functions and helper methods
- Test error handling scenarios and edge cases
- Mock external API calls and hardware dependencies
- For adapters connecting to APIs/devices not reachable by internet, provide example data files to allow testing of functionality without live connections
- Example test structure:
  ```javascript
  describe('AdapterName', () => {
    let adapter;
    
    beforeEach(() => {
      // Setup test adapter instance
    });
    
    test('should initialize correctly', () => {
      // Test adapter initialization
    });
  });
  ```

### Integration Testing

**IMPORTANT**: Use the official `@iobroker/testing` framework for all integration tests. This is the ONLY correct way to test ioBroker adapters.

**Official Documentation**: https://github.com/ioBroker/testing

#### Framework Structure
Integration tests MUST follow this exact pattern:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');

// Define test coordinates or configuration
const TEST_COORDINATES = '52.520008,13.404954'; // Berlin
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        harness = getHarness();
                        
                        // Get adapter object using promisified pattern
                        const obj = await new Promise((res, rej) => {
                            harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                                if (err) return rej(err);
                                res(o);
                            });
                        });
                        
                        if (!obj) {
                            return reject(new Error('Adapter object not found'));
                        }

                        // Configure adapter properties
                        Object.assign(obj.native, {
                            position: TEST_COORDINATES,
                            createCurrently: true,
                            createHourly: true,
                            createDaily: true,
                            // Add other configuration as needed
                        });

                        // Set the updated configuration
                        harness.objects.setObject(obj._id, obj);

                        console.log('✅ Step 1: Configuration written, starting adapter...');
                        
                        // Start adapter and wait
                        await harness.startAdapterAndWait();
                        
                        console.log('✅ Step 2: Adapter started');

                        // Wait for adapter to process data
                        const waitMs = 15000;
                        await wait(waitMs);

                        console.log('🔍 Step 3: Checking states after adapter run...');
                        
                        // Get all states created by adapter
                        const stateIds = await harness.dbConnection.getStateIDs('your-adapter.0.*');
                        
                        console.log(`📊 Found ${stateIds.length} states`);

                        if (stateIds.length > 0) {
                            console.log('✅ Adapter successfully created states');
                            
                            // Show sample of created states
                            const allStates = await new Promise((res, rej) => {
                                harness.states.getStates(stateIds, (err, states) => {
                                    if (err) return rej(err);
                                    res(states || []);
                                });
                            });
                            
                            console.log('📋 Sample states created:');
                            stateIds.slice(0, 5).forEach((stateId, index) => {
                                const state = allStates[index];
                                console.log(`   ${stateId}: ${state && state.val !== undefined ? state.val : 'undefined'}`);
                            });
                            
                            await harness.stopAdapter();
                            resolve(true);
                        } else {
                            console.log('❌ No states were created by the adapter');
                            reject(new Error('Adapter did not create any states'));
                        }
                    } catch (error) {
                        reject(error);
                    }
                });
            }).timeout(40000);
        });
    }
});
```

#### Testing Both Success AND Failure Scenarios

**IMPORTANT**: For every "it works" test, implement corresponding "it doesn't work and fails" tests. This ensures proper error handling and validates that your adapter fails gracefully when expected.

```javascript
// Example: Testing successful configuration
it('should configure and start adapter with valid configuration', function () {
    return new Promise(async (resolve, reject) => {
        // ... successful configuration test as shown above
    });
}).timeout(40000);

// Example: Testing failure scenarios
it('should NOT create daily states when daily is disabled', function () {
    return new Promise(async (resolve, reject) => {
        try {
            harness = getHarness();
            
            console.log('🔍 Step 1: Fetching adapter object...');
            const obj = await new Promise((res, rej) => {
                harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                    if (err) return rej(err);
                    res(o);
                });
            });
            
            if (!obj) {
                return reject(new Error('Adapter object not found'));
            }

            // Configure adapter with daily disabled
            Object.assign(obj.native, {
                position: TEST_COORDINATES,
                createCurrently: true,
                createHourly: true,
                createDaily: false, // Daily is disabled
            });

            harness.objects.setObject(obj._id, obj);

            console.log('✅ Step 1: Configuration written, starting adapter...');
            await harness.startAdapterAndWait();
            console.log('✅ Step 2: Adapter started');

            const waitMs = 15000;
            await wait(waitMs);

            console.log('🔍 Step 3: Checking states after adapter run...');
            
            // Get states and specifically check NO daily states exist
            const stateIds = await harness.dbConnection.getStateIDs('your-adapter.0.*');
            const dailyStates = stateIds.filter(id => id.includes('.forecast.daily.'));
            
            console.log(`📊 Total states: ${stateIds.length}, Daily states found: ${dailyStates.length}`);
            
            if (dailyStates.length === 0) {
                console.log('✅ Correctly NO daily states created when disabled');
                await harness.stopAdapter();
                resolve(true);
            } else {
                console.log('❌ Daily states were created despite being disabled');
                console.log('Daily states found:', dailyStates);
                reject(new Error(`Daily states were created when disabled: ${dailyStates.join(', ')}`));
            }
        } catch (error) {
            reject(error);
        }
    });
}).timeout(40000);
```

#### Using @iobroker/testing Framework Correctly

**Key Features of @iobroker/testing:**
- **Automated Test Environment**: Sets up a complete ioBroker instance for testing
- **Database Management**: Provides clean database state for each test
- **Adapter Lifecycle**: Manages adapter start/stop operations
- **State Validation**: Tools to verify state creation and values
- **Configuration Management**: Easy adapter configuration setup

**Common Integration Test Patterns:**

1. **Basic Adapter Functionality**
```javascript
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Basic adapter functionality', (getHarness) => {
            it('should start and create basic states', async function () {
                const harness = getHarness();
                
                // Configure adapter
                const adapterObj = await harness.objects.getObjectAsync('system.adapter.your-adapter.0');
                adapterObj.native = {
                    // Your adapter configuration
                };
                await harness.objects.setObjectAsync(adapterObj._id, adapterObj);
                
                // Start adapter
                await harness.startAdapterAndWait();
                
                // Wait for processing
                await new Promise(resolve => setTimeout(resolve, 5000));
                
                // Verify states
                const states = await harness.states.getStatesAsync('your-adapter.0.*');
                expect(Object.keys(states).length).toBeGreaterThan(0);
                
                await harness.stopAdapter();
            });
        });
    }
});
```

2. **Configuration Testing**
```javascript
it('should handle different configuration options', async function () {
    const harness = getHarness();
    
    // Test with specific configuration
    const configs = [
        { setting1: true, setting2: 'value1' },
        { setting1: false, setting2: 'value2' },
    ];
    
    for (const config of configs) {
        const adapterObj = await harness.objects.getObjectAsync('system.adapter.your-adapter.0');
        adapterObj.native = config;
        await harness.objects.setObjectAsync(adapterObj._id, adapterObj);
        
        await harness.startAdapterAndWait();
        // Verify behavior for this configuration
        await harness.stopAdapter();
    }
});
```

3. **Error Handling Testing**
```javascript
it('should handle invalid configuration gracefully', async function () {
    const harness = getHarness();
    
    const adapterObj = await harness.objects.getObjectAsync('system.adapter.your-adapter.0');
    adapterObj.native = {
        invalidSetting: 'invalid_value'
    };
    await harness.objects.setObjectAsync(adapterObj._id, adapterObj);
    
    await harness.startAdapterAndWait();
    
    // Verify adapter started but handled error gracefully
    const adapterState = await harness.states.getStateAsync('system.adapter.your-adapter.0.alive');
    expect(adapterState.val).toBe(true);
    
    await harness.stopAdapter();
});
```

## ioBroker Adapter Development Standards

### Logging Best Practices

Use appropriate logging levels:
- `this.log.error()` - Critical errors that prevent functionality
- `this.log.warn()` - Warnings about potential issues
- `this.log.info()` - General information about adapter operation
- `this.log.debug()` - Detailed debugging information

```javascript
// Good logging example
try {
    const data = await this.fetchData();
    this.log.info('Successfully fetched data');
    this.log.debug(`Data received: ${JSON.stringify(data)}`);
} catch (error) {
    this.log.error(`Failed to fetch data: ${error.message}`);
}
```

### State Management

#### Creating States
Always define state objects before setting values:

```javascript
await this.setObjectNotExistsAsync('device.state', {
    type: 'state',
    common: {
        name: 'State Name',
        type: 'number',
        role: 'value',
        read: true,
        write: false,
        unit: '%'
    },
    native: {}
});

await this.setStateAsync('device.state', { val: value, ack: true });
```

#### State Roles and Types
Use appropriate common.role values:
- `value` - General numeric value
- `value.temperature` - Temperature values
- `switch` - Boolean on/off states
- `button` - Trigger buttons
- `indicator` - Status indicators

### Error Handling

Always implement proper error handling:

```javascript
// HTTP requests
try {
    const response = await axios.get(url);
    return response.data;
} catch (error) {
    if (error.response) {
        this.log.error(`HTTP ${error.response.status}: ${error.response.statusText}`);
    } else if (error.request) {
        this.log.error('Network error - no response received');
    } else {
        this.log.error(`Request setup error: ${error.message}`);
    }
    throw error;
}

// Async operations
async function safeOperation() {
    try {
        const result = await riskyOperation();
        return result;
    } catch (error) {
        this.log.error(`Operation failed: ${error.message}`);
        return null;
    }
}
```

### Adapter Lifecycle

#### Proper Startup
```javascript
class AdapterName extends utils.Adapter {
    constructor(options = {}) {
        super({
            ...options,
            name: 'adapter-name',
        });
        
        this.on('ready', this.onReady.bind(this));
        this.on('unload', this.onUnload.bind(this));
    }

    async onReady() {
        this.log.info('Adapter started');
        
        // Initialize adapter functionality
        await this.initializeStates();
        await this.startDataCollection();
    }

    async onUnload(callback) {
        try {
            this.log.info('Adapter stopping');
            
            // Clean up resources
            if (this.updateTimer) {
                clearTimeout(this.updateTimer);
            }
            
            callback();
        } catch (e) {
            callback();
        }
    }
}
```

#### Resource Cleanup
Always clean up resources in the `unload` method:

```javascript
async onUnload(callback) {
    try {
        // Clear timers
        if (this.updateTimer) clearTimeout(this.updateTimer);
        if (this.pollInterval) clearInterval(this.pollInterval);
        
        // Close connections
        if (this.connection) await this.connection.close();
        
        // Stop services
        if (this.service) await this.service.stop();
        
        this.log.info('Cleaned up successfully');
        callback();
    } catch (e) {
        this.log.error(`Error during cleanup: ${e.message}`);
        callback();
    }
}
```

### Configuration Management

Use `io-package.json` for configuration schema:

```json
{
    "common": {
        "adminUI": {
            "config": "json"
        }
    },
    "native": {
        "apiKey": "",
        "updateInterval": 300,
        "enableFeature": false
    }
}
```

Access configuration in adapter code:
```javascript
const apiKey = this.config.apiKey;
const interval = this.config.updateInterval || 300;
```

### Scheduled Adapters

For adapters that run on schedule:

```javascript
// In io-package.json
{
    "common": {
        "mode": "schedule",
        "schedule": "*/15 * * * *"  // Every 15 minutes
    }
}

// In adapter code
async onReady() {
    // Perform scheduled task
    await this.performScheduledTask();
    
    // Stop adapter after task completion
    setTimeout(() => this.terminate(), 1000);
}
```

### HTTP Requests

Use proper HTTP libraries with timeout handling:

```javascript
const axios = require('axios');

// Configure axios instance
this.httpClient = axios.create({
    timeout: 10000,
    headers: {
        'User-Agent': 'ioBroker-Adapter/1.0'
    }
});

// Add request interceptor for logging
this.httpClient.interceptors.request.use(request => {
    this.log.debug(`HTTP Request: ${request.method?.toUpperCase()} ${request.url}`);
    return request;
});

// Add response interceptor for error handling
this.httpClient.interceptors.response.use(
    response => response,
    error => {
        this.log.warn(`HTTP Error: ${error.message}`);
        return Promise.reject(error);
    }
);
```

### Data Validation

Always validate data before processing:

```javascript
function validateData(data) {
    if (!data || typeof data !== 'object') {
        throw new Error('Invalid data format');
    }
    
    if (!data.value || typeof data.value !== 'number') {
        throw new Error('Missing or invalid value');
    }
    
    return true;
}

// Usage
try {
    validateData(receivedData);
    await this.processData(receivedData);
} catch (error) {
    this.log.error(`Data validation failed: ${error.message}`);
}
```

## TypeScript Support

For TypeScript adapters, use proper type definitions:

```typescript
import { Adapter, AdapterConfig } from '@iobroker/adapter-core';

interface AdapterConfigType extends AdapterConfig {
    apiKey: string;
    updateInterval: number;
    enableFeature: boolean;
}

class AdapterName extends Adapter {
    private config: AdapterConfigType;
    
    constructor(options = {}) {
        super({
            ...options,
            name: 'adapter-name',
        });
        
        this.config = this.config as AdapterConfigType;
    }
}
```

## Performance Best Practices

1. **Batch State Updates**
   ```javascript
   // Instead of multiple setState calls
   const states = [
       { id: 'device.state1', val: value1 },
       { id: 'device.state2', val: value2 },
   ];
   
   await this.setStatesAsync(states);
   ```

2. **Efficient Data Processing**
   ```javascript
   // Use Promise.all for parallel operations
   const promises = devices.map(device => this.processDevice(device));
   await Promise.all(promises);
   ```

3. **Memory Management**
   ```javascript
   // Clear large objects when done
   let largeDataset = await this.fetchLargeDataset();
   await this.processData(largeDataset);
   largeDataset = null; // Allow garbage collection
   ```

**Stockmarket Adapter Specific Patterns:**
- Use scheduled mode for periodic stock price updates
- Implement proper API rate limiting and error handling for stock market APIs
- Cache stock symbols validation to avoid repeated API calls
- Structure states hierarchically by stock symbol (e.g., 'AAPL.price', 'AAPL.change')
- Handle market hours and weekend scenarios gracefully
- Validate stock symbols before making API requests
- Use proper number formatting for financial data (price, volume, percentages)