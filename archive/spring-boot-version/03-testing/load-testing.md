# Load Testing

This document provides comprehensive strategies for performance and load testing the First Viscount microservices platform using K6 and JMeter.

## Overview

Load testing ensures the platform can:
- Handle expected user loads
- Scale under peak traffic
- Maintain response time SLAs
- Identify performance bottlenecks
- Validate resource utilization

## K6 Load Testing

### Installation and Setup

```bash
# Install K6 on Ubuntu
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6

# Or using Docker
docker run -i grafana/k6 run - <script.js
```

### Basic Load Test Script

```javascript
// k6/basic-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

// Define custom metrics
const errorRate = new Rate('errors');

// Test configuration
export const options = {
  stages: [
    { duration: '2m', target: 10 },   // Ramp up to 10 users
    { duration: '5m', target: 50 },   // Stay at 50 users
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 0 },    // Ramp down to 0 users
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests under 500ms
    http_req_failed: ['rate<0.1'],     // Error rate under 10%
    errors: ['rate<0.1'],              // Custom error rate under 10%
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';
const AUTH_TOKEN = __ENV.AUTH_TOKEN || '';

export function setup() {
  // Setup code - login and get auth token
  const loginRes = http.post(`${BASE_URL}/api/v1/auth/login`, JSON.stringify({
    username: 'testuser',
    password: 'testpass'
  }), {
    headers: { 'Content-Type': 'application/json' },
  });
  
  const authToken = loginRes.json('accessToken');
  return { authToken };
}

export default function(data) {
  const params = {
    headers: {
      'Authorization': `Bearer ${data.authToken}`,
      'Content-Type': 'application/json',
    },
  };

  // Scenario 1: Browse products
  const productsRes = http.get(`${BASE_URL}/api/v1/products?page=1&size=20`, params);
  check(productsRes, {
    'products status is 200': (r) => r.status === 200,
    'products returned': (r) => JSON.parse(r.body).content.length > 0,
  });
  errorRate.add(productsRes.status !== 200);
  
  sleep(1);
  
  // Scenario 2: View product details
  if (productsRes.status === 200) {
    const products = JSON.parse(productsRes.body).content;
    const randomProduct = products[Math.floor(Math.random() * products.length)];
    
    const productRes = http.get(`${BASE_URL}/api/v1/products/${randomProduct.id}`, params);
    check(productRes, {
      'product details status is 200': (r) => r.status === 200,
    });
  }
  
  sleep(2);
  
  // Scenario 3: Add to cart (30% of users)
  if (Math.random() < 0.3) {
    const cartRes = http.post(
      `${BASE_URL}/api/v1/cart/items`,
      JSON.stringify({
        productId: 1,
        quantity: Math.floor(Math.random() * 3) + 1
      }),
      params
    );
    check(cartRes, {
      'add to cart status is 201': (r) => r.status === 201,
    });
  }
  
  sleep(1);
}

export function teardown(data) {
  // Cleanup code
  console.log('Test completed');
}
```

### Complex E-Commerce Scenarios

```javascript
// k6/ecommerce-scenarios.js
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { SharedArray } from 'k6/data';
import { randomItem } from 'https://jslib.k6.io/k6-utils/1.2.0/index.js';

// Load test data
const products = new SharedArray('products', function() {
  return JSON.parse(open('./test-data/products.json'));
});

const users = new SharedArray('users', function() {
  return JSON.parse(open('./test-data/users.json'));
});

export const options = {
  scenarios: {
    browse_products: {
      executor: 'constant-vus',
      vus: 50,
      duration: '10m',
      exec: 'browseProducts',
    },
    search_products: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 20 },
        { duration: '5m', target: 20 },
        { duration: '2m', target: 0 },
      ],
      exec: 'searchProducts',
    },
    checkout_flow: {
      executor: 'ramping-arrival-rate',
      startRate: 1,
      timeUnit: '1s',
      preAllocatedVUs: 50,
      stages: [
        { duration: '2m', target: 5 },
        { duration: '5m', target: 10 },
        { duration: '3m', target: 5 },
      ],
      exec: 'checkoutFlow',
    },
  },
  thresholds: {
    'http_req_duration{scenario:browse_products}': ['p(95)<300'],
    'http_req_duration{scenario:search_products}': ['p(95)<500'],
    'http_req_duration{scenario:checkout_flow}': ['p(95)<1000'],
    'http_req_failed': ['rate<0.05'],
    'checks': ['rate>0.95'],
  },
};

// Scenario: Browse Products
export function browseProducts() {
  const user = randomItem(users);
  const params = {
    headers: {
      'Authorization': `Bearer ${user.token}`,
    },
    tags: { scenario: 'browse_products' },
  };
  
  group('Browse Product Catalog', () => {
    // List categories
    const categoriesRes = http.get(`${BASE_URL}/api/v1/categories`, params);
    check(categoriesRes, {
      'categories loaded': (r) => r.status === 200,
    });
    
    sleep(randomIntBetween(1, 3));
    
    // Browse products in category
    const categories = JSON.parse(categoriesRes.body);
    const category = randomItem(categories);
    
    const productsRes = http.get(
      `${BASE_URL}/api/v1/products?category=${category.id}&page=1&size=20`,
      params
    );
    check(productsRes, {
      'products loaded': (r) => r.status === 200,
      'products in category': (r) => JSON.parse(r.body).content.length > 0,
    });
    
    sleep(randomIntBetween(2, 5));
    
    // View product details
    if (productsRes.status === 200) {
      const products = JSON.parse(productsRes.body).content;
      const product = randomItem(products);
      
      const detailsRes = http.get(`${BASE_URL}/api/v1/products/${product.id}`, params);
      check(detailsRes, {
        'product details loaded': (r) => r.status === 200,
      });
      
      // Check inventory
      const inventoryRes = http.get(`${BASE_URL}/api/v1/inventory/${product.id}`, params);
      check(inventoryRes, {
        'inventory checked': (r) => r.status === 200,
      });
    }
  });
}

// Scenario: Search Products
export function searchProducts() {
  const searchTerms = ['laptop', 'coffee', 'shirt', 'book', 'phone', 'desk'];
  const searchTerm = randomItem(searchTerms);
  
  const params = {
    tags: { scenario: 'search_products' },
  };
  
  group('Product Search', () => {
    const searchRes = http.get(
      `${BASE_URL}/api/v1/products/search?q=${searchTerm}&page=1&size=20`,
      params
    );
    
    check(searchRes, {
      'search completed': (r) => r.status === 200,
      'search has results': (r) => {
        const body = JSON.parse(r.body);
        return body.content && body.content.length > 0;
      },
      'search response time OK': (r) => r.timings.duration < 500,
    });
    
    sleep(randomIntBetween(1, 2));
    
    // Filter search results
    if (searchRes.status === 200) {
      const filterRes = http.get(
        `${BASE_URL}/api/v1/products/search?q=${searchTerm}&minPrice=10&maxPrice=1000&sort=price`,
        params
      );
      
      check(filterRes, {
        'filtered search completed': (r) => r.status === 200,
      });
    }
  });
}

// Scenario: Complete Checkout Flow
export function checkoutFlow() {
  const user = randomItem(users);
  const params = {
    headers: {
      'Authorization': `Bearer ${user.token}`,
      'Content-Type': 'application/json',
    },
    tags: { scenario: 'checkout_flow' },
  };
  
  group('Checkout Process', () => {
    // Step 1: Add items to cart
    const cartItems = [];
    const numItems = randomIntBetween(1, 5);
    
    for (let i = 0; i < numItems; i++) {
      const product = randomItem(products);
      const addToCartRes = http.post(
        `${BASE_URL}/api/v1/cart/items`,
        JSON.stringify({
          productId: product.id,
          quantity: randomIntBetween(1, 3)
        }),
        params
      );
      
      check(addToCartRes, {
        'item added to cart': (r) => r.status === 201,
      });
      
      if (addToCartRes.status === 201) {
        cartItems.push(JSON.parse(addToCartRes.body));
      }
      
      sleep(0.5);
    }
    
    // Step 2: View cart
    const cartRes = http.get(`${BASE_URL}/api/v1/cart`, params);
    check(cartRes, {
      'cart retrieved': (r) => r.status === 200,
      'cart has items': (r) => JSON.parse(r.body).items.length > 0,
    });
    
    sleep(2);
    
    // Step 3: Create order
    const orderPayload = {
      items: cartItems.map(item => ({
        productId: item.productId,
        quantity: item.quantity
      })),
      shippingAddress: {
        street: '123 Test St',
        city: 'Seattle',
        state: 'WA',
        zipCode: '98101',
        country: 'USA'
      },
      paymentMethod: 'CREDIT_CARD',
      cardNumber: '4111111111111111',
      cvv: '123',
      expiryMonth: 12,
      expiryYear: 2025
    };
    
    const orderRes = http.post(
      `${BASE_URL}/api/v1/orders`,
      JSON.stringify(orderPayload),
      params
    );
    
    check(orderRes, {
      'order created': (r) => r.status === 201,
      'order has ID': (r) => JSON.parse(r.body).orderId !== undefined,
    });
    
    if (orderRes.status === 201) {
      const order = JSON.parse(orderRes.body);
      
      // Step 4: Check order status
      sleep(3);
      
      const statusRes = http.get(`${BASE_URL}/api/v1/orders/${order.orderId}`, params);
      check(statusRes, {
        'order status retrieved': (r) => r.status === 200,
        'order is processing': (r) => {
          const status = JSON.parse(r.body).status;
          return status === 'PROCESSING' || status === 'CONFIRMED';
        },
      });
    }
  });
}

function randomIntBetween(min, max) {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}
```

### Spike Testing

```javascript
// k6/spike-test.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '10s', target: 0 },     // Start with 0 users
    { duration: '10s', target: 100 },   // Spike to 100 users
    { duration: '30s', target: 100 },   // Stay at 100 users
    { duration: '10s', target: 1000 },  // Spike to 1000 users
    { duration: '30s', target: 1000 },  // Stay at 1000 users
    { duration: '10s', target: 100 },   // Drop to 100 users
    { duration: '30s', target: 100 },   // Stay at 100 users
    { duration: '10s', target: 0 },     // Ramp down to 0
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'], // 95% of requests under 2s during spike
    http_req_failed: ['rate<0.2'],     // Error rate under 20% during spike
  },
};

export default function() {
  const responses = http.batch([
    ['GET', `${BASE_URL}/api/v1/products?page=1&size=10`],
    ['GET', `${BASE_URL}/api/v1/categories`],
    ['GET', `${BASE_URL}/api/v1/products/bestsellers`],
  ]);
  
  responses.forEach((res, idx) => {
    check(res, {
      [`request ${idx} status is 200`]: (r) => r.status === 200,
    });
  });
}
```

### Stress Testing

```javascript
// k6/stress-test.js
export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Below normal load
    { duration: '5m', target: 200 },   // Normal load
    { duration: '2m', target: 300 },   // Around breaking point
    { duration: '5m', target: 400 },   // Beyond breaking point
    { duration: '2m', target: 500 },   // Peak load
    { duration: '10m', target: 0 },    // Recovery stage
  ],
};
```

### Soak Testing

```javascript
// k6/soak-test.js
export const options = {
  stages: [
    { duration: '5m', target: 100 },   // Ramp up
    { duration: '4h', target: 100 },   // Stay at 100 users for 4 hours
    { duration: '5m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<1000'], // Maintain performance over time
    http_req_failed: ['rate<0.01'],    // Very low error rate
  },
};
```

## JMeter Load Testing

### JMeter Test Plan Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0" jmeter="5.5">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="First Viscount Load Test" enabled="true">
      <stringProp name="TestPlan.comments">E-commerce platform load test</stringProp>
      <boolProp name="TestPlan.functional_mode">false</boolProp>
      <boolProp name="TestPlan.tearDown_on_shutdown">true</boolProp>
      <boolProp name="TestPlan.serialize_threadgroups">false</boolProp>
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments">
        <collectionProp name="Arguments.arguments">
          <elementProp name="BASE_URL" elementType="Argument">
            <stringProp name="Argument.name">BASE_URL</stringProp>
            <stringProp name="Argument.value">${__P(base.url,http://localhost:8080)}</stringProp>
          </elementProp>
          <elementProp name="USERS" elementType="Argument">
            <stringProp name="Argument.name">USERS</stringProp>
            <stringProp name="Argument.value">${__P(users,100)}</stringProp>
          </elementProp>
        </collectionProp>
      </elementProp>
    </TestPlan>
    <hashTree>
      
      <!-- HTTP Request Defaults -->
      <ConfigTestElement guiclass="HttpDefaultsGui" testclass="ConfigTestElement" testname="HTTP Request Defaults" enabled="true">
        <elementProp name="HTTPsampler.Arguments" elementType="Arguments"/>
        <stringProp name="HTTPSampler.domain">${BASE_URL}</stringProp>
        <stringProp name="HTTPSampler.protocol">http</stringProp>
        <stringProp name="HTTPSampler.contentEncoding">UTF-8</stringProp>
      </ConfigTestElement>
      <hashTree/>
      
      <!-- HTTP Header Manager -->
      <HeaderManager guiclass="HeaderPanel" testclass="HeaderManager" testname="HTTP Header Manager" enabled="true">
        <collectionProp name="HeaderManager.headers">
          <elementProp name="">
            <stringProp name="Header.name">Content-Type</stringProp>
            <stringProp name="Header.value">application/json</stringProp>
          </elementProp>
          <elementProp name="">
            <stringProp name="Header.name">Accept</stringProp>
            <stringProp name="Header.value">application/json</stringProp>
          </elementProp>
        </collectionProp>
      </HeaderManager>
      <hashTree/>
      
      <!-- User Defined Variables -->
      <Arguments guiclass="ArgumentsPanel" testclass="Arguments" testname="User Defined Variables" enabled="true">
        <collectionProp name="Arguments.arguments">
          <elementProp name="THINK_TIME" elementType="Argument">
            <stringProp name="Argument.name">THINK_TIME</stringProp>
            <stringProp name="Argument.value">2000</stringProp>
          </elementProp>
        </collectionProp>
      </Arguments>
      <hashTree/>
      
      <!-- Thread Group: Browse Products -->
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Browse Products" enabled="true">
        <stringProp name="ThreadGroup.on_sample_error">continue</stringProp>
        <elementProp name="ThreadGroup.main_controller" elementType="LoopController">
          <boolProp name="LoopController.continue_forever">false</boolProp>
          <intProp name="LoopController.loops">10</intProp>
        </elementProp>
        <stringProp name="ThreadGroup.num_threads">${USERS}</stringProp>
        <stringProp name="ThreadGroup.ramp_time">60</stringProp>
        <boolProp name="ThreadGroup.scheduler">true</boolProp>
        <stringProp name="ThreadGroup.duration">600</stringProp>
        <stringProp name="ThreadGroup.delay">0</stringProp>
      </ThreadGroup>
      <hashTree>
        
        <!-- Login Once Only Controller -->
        <OnceOnlyController guiclass="OnceOnlyControllerGui" testclass="OnceOnlyController" testname="Login Once" enabled="true"/>
        <hashTree>
          <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="Login" enabled="true">
            <boolProp name="HTTPSampler.postBodyRaw">true</boolProp>
            <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
              <collectionProp name="Arguments.arguments">
                <elementProp name="" elementType="HTTPArgument">
                  <boolProp name="HTTPArgument.always_encode">false</boolProp>
                  <stringProp name="Argument.value">{
  "username": "user${__threadNum}",
  "password": "password123"
}</stringProp>
                  <stringProp name="Argument.metadata">=</stringProp>
                </elementProp>
              </collectionProp>
            </elementProp>
            <stringProp name="HTTPSampler.path">/api/v1/auth/login</stringProp>
            <stringProp name="HTTPSampler.method">POST</stringProp>
          </HTTPSamplerProxy>
          <hashTree>
            <!-- JSON Extractor for Auth Token -->
            <JSONPostProcessor guiclass="JSONPostProcessorGui" testclass="JSONPostProcessor" testname="Extract Auth Token" enabled="true">
              <stringProp name="JSONPostProcessor.referenceNames">authToken</stringProp>
              <stringProp name="JSONPostProcessor.jsonPathExprs">$.accessToken</stringProp>
            </JSONPostProcessor>
            <hashTree/>
          </hashTree>
        </hashTree>
        
        <!-- Simple Controller: Product Browsing -->
        <GenericController guiclass="LogicControllerGui" testclass="GenericController" testname="Product Browsing Flow" enabled="true"/>
        <hashTree>
          
          <!-- Get Product List -->
          <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="Get Products" enabled="true">
            <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
              <collectionProp name="Arguments.arguments">
                <elementProp name="page" elementType="HTTPArgument">
                  <boolProp name="HTTPArgument.always_encode">false</boolProp>
                  <stringProp name="Argument.name">page</stringProp>
                  <stringProp name="Argument.value">${__Random(1,10)}</stringProp>
                </elementProp>
                <elementProp name="size" elementType="HTTPArgument">
                  <boolProp name="HTTPArgument.always_encode">false</boolProp>
                  <stringProp name="Argument.name">size</stringProp>
                  <stringProp name="Argument.value">20</stringProp>
                </elementProp>
              </collectionProp>
            </elementProp>
            <stringProp name="HTTPSampler.path">/api/v1/products</stringProp>
            <stringProp name="HTTPSampler.method">GET</stringProp>
          </HTTPSamplerProxy>
          <hashTree>
            <!-- Extract Product IDs -->
            <JSONPostProcessor guiclass="JSONPostProcessorGui" testclass="JSONPostProcessor" testname="Extract Product IDs" enabled="true">
              <stringProp name="JSONPostProcessor.referenceNames">productIds</stringProp>
              <stringProp name="JSONPostProcessor.jsonPathExprs">$.content[*].id</stringProp>
              <stringProp name="JSONPostProcessor.match_numbers">-1</stringProp>
            </JSONPostProcessor>
            <hashTree/>
            
            <!-- Response Assertion -->
            <ResponseAssertion guiclass="AssertionGui" testclass="ResponseAssertion" testname="Response Assertion" enabled="true">
              <collectionProp name="Asserion.test_strings">
                <stringProp name="49586">200</stringProp>
              </collectionProp>
              <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
            </ResponseAssertion>
            <hashTree/>
          </hashTree>
          
          <!-- Think Time -->
          <ConstantTimer guiclass="ConstantTimerGui" testclass="ConstantTimer" testname="Think Time" enabled="true">
            <stringProp name="ConstantTimer.delay">${__Random(1000,3000)}</stringProp>
          </ConstantTimer>
          <hashTree/>
          
          <!-- View Product Details -->
          <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="View Product Details" enabled="true">
            <elementProp name="HTTPsampler.Arguments" elementType="Arguments"/>
            <stringProp name="HTTPSampler.path">/api/v1/products/${productIds_${__Random(1,${productIds_matchNr})}}</stringProp>
            <stringProp name="HTTPSampler.method">GET</stringProp>
          </HTTPSamplerProxy>
          <hashTree/>
          
        </hashTree>
      </hashTree>
      
      <!-- Aggregate Report -->
      <ResultCollector guiclass="StatVisualizer" testclass="ResultCollector" testname="Aggregate Report" enabled="true">
        <boolProp name="ResultCollector.error_logging">false</boolProp>
        <objProp>
          <name>saveConfig</name>
          <value class="SampleSaveConfiguration">
            <time>true</time>
            <latency>true</latency>
            <timestamp>true</timestamp>
            <success>true</success>
            <label>true</label>
            <code>true</code>
            <message>true</message>
            <threadName>true</threadName>
            <dataType>true</dataType>
            <encoding>false</encoding>
            <assertions>true</assertions>
            <subresults>true</subresults>
            <responseData>false</responseData>
            <samplerData>false</samplerData>
            <xml>false</xml>
            <fieldNames>true</fieldNames>
            <responseHeaders>false</responseHeaders>
            <requestHeaders>false</requestHeaders>
            <responseDataOnError>false</responseDataOnError>
            <saveAssertionResultsFailureMessage>true</saveAssertionResultsFailureMessage>
            <assertionsResultsToSave>0</assertionsResultsToSave>
            <bytes>true</bytes>
            <sentBytes>true</sentBytes>
            <url>true</url>
            <threadCounts>true</threadCounts>
            <idleTime>true</idleTime>
            <connectTime>true</connectTime>
          </value>
        </objProp>
        <stringProp name="filename"></stringProp>
      </ResultCollector>
      <hashTree/>
      
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

### JMeter Command Line Execution

```bash
#!/bin/bash
# run-jmeter-test.sh

# Set JMeter home
export JMETER_HOME=/opt/apache-jmeter-5.5

# Test parameters
TEST_PLAN="ecommerce-load-test.jmx"
RESULTS_FILE="results_$(date +%Y%m%d_%H%M%S).jtl"
REPORT_DIR="report_$(date +%Y%m%d_%H%M%S)"

# Run test
$JMETER_HOME/bin/jmeter -n \
  -t $TEST_PLAN \
  -l $RESULTS_FILE \
  -e \
  -o $REPORT_DIR \
  -Jbase.url=http://api.firstviscount.com \
  -Jusers=100 \
  -Jduration=600 \
  -Jrampup=60

# Generate HTML report
$JMETER_HOME/bin/jmeter -g $RESULTS_FILE -o $REPORT_DIR/dashboard

echo "Test completed. Results in $REPORT_DIR"
```

## Custom Metrics and Monitoring

### K6 Custom Metrics

```javascript
// k6/custom-metrics.js
import http from 'k6/http';
import { Trend, Counter, Gauge, Rate } from 'k6/metrics';

// Custom metrics
const orderProcessingTime = new Trend('order_processing_time');
const ordersCreated = new Counter('orders_created');
const inventoryLevel = new Gauge('inventory_level');
const orderFailureRate = new Rate('order_failure_rate');

export default function() {
  // Track order creation
  const startTime = new Date();
  const orderRes = http.post(
    `${BASE_URL}/api/v1/orders`,
    JSON.stringify({
      items: [{ productId: 1, quantity: 1 }],
      // ... other order data
    })
  );
  
  const processingTime = new Date() - startTime;
  
  if (orderRes.status === 201) {
    orderProcessingTime.add(processingTime);
    ordersCreated.add(1);
    orderFailureRate.add(false);
  } else {
    orderFailureRate.add(true);
  }
  
  // Track inventory levels
  const inventoryRes = http.get(`${BASE_URL}/api/v1/inventory/1`);
  if (inventoryRes.status === 200) {
    const stock = JSON.parse(inventoryRes.body).availableQuantity;
    inventoryLevel.add(stock);
  }
}
```

### Prometheus Integration

```javascript
// k6/prometheus-remote-write.js
export const options = {
  scenarios: {
    contacts: {
      executor: 'constant-vus',
      vus: 10,
      duration: '30s',
    },
  },
  // Prometheus remote write configuration
  prometheus: {
    url: 'http://localhost:9090/api/v1/write',
    headers: {
      'X-Prometheus-Remote-Write-Version': '0.1.0',
    },
    insecureSkipVerify: true,
  },
};
```

## Performance Testing in CI/CD

### GitHub Actions Integration

```yaml
name: Performance Tests

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM

jobs:
  k6-load-test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Start services
      run: docker-compose -f docker-compose.test.yml up -d
    
    - name: Wait for services
      run: |
        timeout 60 bash -c 'until curl -f http://localhost:8080/health; do sleep 1; done'
    
    - name: Run K6 tests
      uses: grafana/k6-action@v0.3.0
      with:
        filename: k6/load-test.js
        flags: --out json=results.json
      env:
        BASE_URL: http://localhost:8080
    
    - name: Upload results
      uses: actions/upload-artifact@v3
      with:
        name: k6-results
        path: results.json
    
    - name: Check thresholds
      run: |
        # Parse results and check against SLAs
        python scripts/check-performance-thresholds.py results.json
    
    - name: Comment PR
      if: github.event_name == 'pull_request'
      uses: actions/github-script@v6
      with:
        script: |
          const results = require('./results.json');
          const comment = `## Performance Test Results
          
          - **Avg Response Time**: ${results.metrics.http_req_duration.avg}ms
          - **95th Percentile**: ${results.metrics.http_req_duration.p95}ms
          - **Error Rate**: ${results.metrics.http_req_failed.rate}%
          - **RPS**: ${results.metrics.http_reqs.rate}
          
          ${results.thresholds_passed ? '✅ All thresholds passed' : '❌ Some thresholds failed'}`;
          
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: comment
          });
```

### Jenkins Pipeline

```groovy
pipeline {
    agent any
    
    parameters {
        string(name: 'USERS', defaultValue: '100', description: 'Number of virtual users')
        string(name: 'DURATION', defaultValue: '600', description: 'Test duration in seconds')
        string(name: 'TARGET_ENV', defaultValue: 'staging', description: 'Target environment')
    }
    
    stages {
        stage('Setup') {
            steps {
                sh 'docker pull grafana/k6'
                sh 'docker pull justb4/jmeter'
            }
        }
        
        stage('K6 Load Test') {
            steps {
                script {
                    sh """
                        docker run --rm \
                            -v ${WORKSPACE}/k6:/scripts \
                            -e BASE_URL=https://${params.TARGET_ENV}.firstviscount.com \
                            grafana/k6 run \
                            --vus ${params.USERS} \
                            --duration ${params.DURATION}s \
                            --out influxdb=http://influxdb:8086/k6 \
                            /scripts/load-test.js
                    """
                }
            }
        }
        
        stage('JMeter Test') {
            steps {
                sh """
                    docker run --rm \
                        -v ${WORKSPACE}/jmeter:/test \
                        justb4/jmeter \
                        -n -t /test/load-test.jmx \
                        -Jusers=${params.USERS} \
                        -Jduration=${params.DURATION}
                """
            }
        }
        
        stage('Generate Reports') {
            steps {
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'jmeter/report',
                    reportFiles: 'index.html',
                    reportName: 'JMeter Report'
                ])
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: '**/results.jtl', fingerprint: true
        }
        failure {
            emailext (
                subject: "Performance Test Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: "Performance thresholds not met. Check the report for details.",
                to: 'platform-team@firstviscount.com'
            )
        }
    }
}
```

## Load Test Data Generation

### Realistic Test Data

```javascript
// k6/data-generator.js
import { SharedArray } from 'k6/data';
import { randomString, randomIntBetween } from 'https://jslib.k6.io/k6-utils/1.2.0/index.js';

export function generateTestUsers(count) {
  const users = [];
  for (let i = 0; i < count; i++) {
    users.push({
      username: `user_${i}_${randomString(8)}`,
      email: `user${i}@loadtest.com`,
      password: 'LoadTest123!',
      firstName: randomItem(['John', 'Jane', 'Bob', 'Alice']),
      lastName: randomItem(['Smith', 'Johnson', 'Williams', 'Brown']),
    });
  }
  return users;
}

export function generateProducts(count) {
  const categories = ['Electronics', 'Clothing', 'Books', 'Home', 'Sports'];
  const adjectives = ['Premium', 'Deluxe', 'Professional', 'Essential', 'Ultra'];
  const products = [];
  
  for (let i = 0; i < count; i++) {
    products.push({
      name: `${randomItem(adjectives)} Product ${i}`,
      price: randomIntBetween(10, 1000),
      category: randomItem(categories),
      sku: `SKU-${randomString(10).toUpperCase()}`,
      inventory: randomIntBetween(0, 1000),
    });
  }
  return products;
}

export function generateOrders(users, products, count) {
  const orders = [];
  for (let i = 0; i < count; i++) {
    const itemCount = randomIntBetween(1, 5);
    const items = [];
    
    for (let j = 0; j < itemCount; j++) {
      items.push({
        product: randomItem(products),
        quantity: randomIntBetween(1, 3),
      });
    }
    
    orders.push({
      user: randomItem(users),
      items: items,
      shippingAddress: generateAddress(),
      paymentMethod: randomItem(['CREDIT_CARD', 'DEBIT_CARD', 'PAYPAL']),
    });
  }
  return orders;
}

function generateAddress() {
  return {
    street: `${randomIntBetween(1, 9999)} ${randomItem(['Main', 'Oak', 'Elm', 'Park'])} St`,
    city: randomItem(['Seattle', 'Portland', 'San Francisco', 'Los Angeles']),
    state: randomItem(['WA', 'OR', 'CA']),
    zipCode: randomIntBetween(10000, 99999).toString(),
    country: 'USA',
  };
}
```

## Result Analysis

### Performance Metrics Script

```python
#!/usr/bin/env python3
# scripts/analyze-performance.py

import json
import sys
import pandas as pd
import matplotlib.pyplot as plt

def analyze_k6_results(filename):
    with open(filename, 'r') as f:
        data = json.load(f)
    
    metrics = data['metrics']
    
    # Response time analysis
    http_duration = metrics['http_req_duration']
    print(f"Response Time Analysis:")
    print(f"  Average: {http_duration['avg']:.2f}ms")
    print(f"  Median: {http_duration['med']:.2f}ms")
    print(f"  95th percentile: {http_duration['p(95)']:.2f}ms")
    print(f"  99th percentile: {http_duration['p(99)']:.2f}ms")
    
    # Error rate
    error_rate = metrics['http_req_failed']['rate'] * 100
    print(f"\nError Rate: {error_rate:.2f}%")
    
    # Throughput
    rps = metrics['http_reqs']['rate']
    print(f"Requests per second: {rps:.2f}")
    
    # Check SLAs
    sla_violations = []
    if http_duration['p(95)'] > 500:
        sla_violations.append("95th percentile response time exceeds 500ms")
    if error_rate > 1:
        sla_violations.append("Error rate exceeds 1%")
    
    if sla_violations:
        print("\n⚠️ SLA Violations:")
        for violation in sla_violations:
            print(f"  - {violation}")
        sys.exit(1)
    else:
        print("\n✅ All SLAs met!")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: python analyze-performance.py <results.json>")
        sys.exit(1)
    
    analyze_k6_results(sys.argv[1])
```

## Best Practices

1. **Realistic Test Data**: Use production-like data volumes and patterns
2. **Gradual Load Increase**: Always ramp up load gradually
3. **Think Time**: Include realistic user think time between actions
4. **Test Environment**: Use dedicated test environment similar to production
5. **Monitor Everything**: Track application metrics alongside load test metrics
6. **Regular Testing**: Run performance tests regularly, not just before releases
7. **Baseline Establishment**: Create performance baselines for comparison
8. **Test Different Scenarios**: Include normal, peak, and stress scenarios

---
*Last Updated: January 2025*