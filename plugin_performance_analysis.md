# Plugin Performance Optimization Analysis

## Current State Analysis

### Bundle Size & Dependencies
- **Main bundle size**: 252KB (minified)
- **Source map size**: 792KB
- **Current bundle contents**:
  - React (18.2.0) - ~45KB gzipped
  - React DOM - ~130KB gzipped
  - Supabase client (2.39.1) - ~80KB gzipped
  - Additional dependencies for animations, fade effects

### Architecture Overview
The plugin is a React-based widget that:
1. Loads as a third-party script on customer landing pages
2. Connects to Supabase backend for data operations
3. Renders interactive Q&A components with custom styling
4. Handles user authentication and answer persistence

## Performance Issues Identified

### 1. Bundle Size (HIGH IMPACT)
**Problem**: 252KB bundle is significantly large for a plugin
- Contains full React runtime
- Includes entire Supabase client library with unused features
- Multiple animation libraries bundled

**SEO Impact**: 
- Increases Time to Interactive (TTI)
- Affects Largest Contentful Paint (LCP)
- May trigger Core Web Vitals warnings

### 2. JavaScript Blocking (HIGH IMPACT)
**Problem**: Plugin loads synchronously, potentially blocking page rendering
```javascript
// Current loading approach blocks DOM parsing
<script src="main.mento.js"></script>
```

### 3. Network Requests (MEDIUM IMPACT)
**Problem**: Multiple API calls during initialization
- Authentication requests
- Question fetching
- User validation
- Panel configuration loading

### 4. Runtime Performance (MEDIUM IMPACT)
**Problem**: React overhead for simple interactions
- Virtual DOM reconciliation for basic UI updates
- Event listeners and state management overhead

## Optimization Recommendations

### 1. Bundle Size Reduction (Priority: HIGH)

#### A. Tree Shaking & Code Splitting
```javascript
// Split into core and features
const coreBundle = {
  // Essential functionality only
  size: "~50KB",
  contains: ["DOM manipulation", "Basic API client", "Core UI"]
};

const featureModules = {
  // Load on demand
  animations: "~15KB", 
  authentication: "~20KB",
  analytics: "~10KB"
};
```

#### B. Replace Heavy Dependencies
- **React → Vanilla JS/Lit**: Reduce from 45KB to ~5KB
- **Supabase Client → Custom API Client**: Reduce from 80KB to ~15KB
- **Animation Libraries → CSS Animations**: Reduce from 15KB to ~2KB

#### C. Implement Module Federation
```javascript
// Load only required modules
const loadModule = async (moduleName) => {
  return await import(/* webpackChunkName: "[request]" */ `./modules/${moduleName}`);
};
```

### 2. Loading Strategy Optimization (Priority: HIGH)

#### A. Implement Async Loading
```html
<!-- Non-blocking script loading -->
<script>
(function() {
  const script = document.createElement('script');
  script.src = 'main.mento.js';
  script.async = true;
  script.defer = true;
  document.head.appendChild(script);
})();
</script>
```

#### B. Progressive Enhancement
```javascript
// 1. Load minimal UI shell first (5KB)
// 2. Enhance with functionality progressively
// 3. Load features on user interaction

const PluginShell = {
  render: () => {
    // Render static HTML first
    document.body.insertAdjacentHTML('beforeend', staticHTML);
  },
  enhance: async () => {
    // Load interactive features on demand
    const { interactivity } = await import('./features/interactivity.js');
    interactivity.init();
  }
};
```

### 3. Caching Strategy (Priority: HIGH)

#### A. Implement Service Worker
```javascript
// Cache static assets and API responses
const CACHE_NAME = 'mento-plugin-v1';
const STATIC_CACHE = [
  '/main.mento.js',
  '/styles.css',
  '/icons.svg'
];

self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/api/questions/')) {
    // Cache questions for 5 minutes
    event.respondWith(cacheFirst(event.request, 300));
  }
});
```

#### B. Browser Caching Headers
```javascript
// Set appropriate cache headers
const cacheHeaders = {
  'Cache-Control': 'public, max-age=31536000', // 1 year for static assets
  'ETag': 'generated-hash-for-version'
};
```

### 4. API Optimization (Priority: MEDIUM)

#### A. Request Batching
```javascript
// Batch multiple API calls
const batchRequests = async (requests) => {
  return await Promise.all([
    fetchUser(),
    fetchQuestions(), 
    fetchPanelConfig()
  ]);
};
```

#### B. Response Compression
```javascript
// Enable gzip/brotli compression
const apiOptions = {
  headers: {
    'Accept-Encoding': 'gzip, deflate, br'
  }
};
```

### 5. Framework Replacement (Priority: HIGH)

#### A. Vanilla JS Implementation
```javascript
// Replace React with vanilla JS
class PluginWidget {
  constructor(config) {
    this.config = config;
    this.element = null;
  }
  
  render() {
    // Direct DOM manipulation
    const template = `
      <div class="mento-widget">
        <div class="question">${this.config.question}</div>
        <div class="answers">${this.renderAnswers()}</div>
      </div>
    `;
    
    const container = document.createElement('div');
    container.innerHTML = template;
    this.element = container.firstElementChild;
    
    document.body.appendChild(this.element);
    this.bindEvents();
  }
  
  bindEvents() {
    // Event delegation for better performance
    this.element.addEventListener('click', this.handleClick.bind(this));
  }
}
```

#### B. Web Components Alternative
```javascript
// Use native Web Components
class MentoWidget extends HTMLElement {
  connectedCallback() {
    this.innerHTML = `
      <style>:host { /* isolated styles */ }</style>
      <div class="widget-content">
        <slot name="question"></slot>
        <slot name="answers"></slot>
      </div>
    `;
  }
}

customElements.define('mento-widget', MentoWidget);
```

### 6. CSS Optimization (Priority: MEDIUM)

#### A. Critical CSS Inlining
```javascript
// Inline critical CSS to prevent FOUC
const criticalCSS = `
  .mento-widget { 
    position: fixed; 
    bottom: 20px; 
    right: 20px; 
    z-index: 999999; 
  }
`;

const style = document.createElement('style');
style.textContent = criticalCSS;
document.head.appendChild(style);
```

#### B. CSS-in-JS Removal
```javascript
// Replace runtime CSS generation with static CSS
// Before: 5KB runtime overhead
// After: 2KB static CSS file
```

### 7. Performance Monitoring (Priority: MEDIUM)

#### A. Core Web Vitals Tracking
```javascript
// Monitor impact on customer sites
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.entryType === 'largest-contentful-paint') {
      // Track LCP impact
      analytics.track('lcp_impact', entry.startTime);
    }
  }
});

observer.observe({entryTypes: ['largest-contentful-paint']});
```

#### B. Bundle Analysis Integration
```javascript
// Continuous bundle size monitoring
const bundleMetrics = {
  size: bundle.size,
  gzipSize: bundle.gzipSize,
  dependencies: bundle.dependencies.length,
  timestamp: Date.now()
};
```

## Implementation Roadmap

### Phase 1: Quick Wins (1-2 weeks)
1. ✅ Implement async loading
2. ✅ Add compression and caching headers
3. ✅ Optimize API request batching
4. ✅ Remove unused Supabase features

**Expected Impact**: 30-40% size reduction, 50% faster TTI

### Phase 2: Architecture Changes (3-4 weeks)
1. ✅ Replace React with vanilla JS/Web Components
2. ✅ Implement custom lightweight API client
3. ✅ Add progressive enhancement
4. ✅ Create modular loading system

**Expected Impact**: 70-80% size reduction, 80% faster TTI

### Phase 3: Advanced Optimizations (2-3 weeks)
1. ✅ Implement Service Worker caching
2. ✅ Add performance monitoring
3. ✅ Optimize animation performance
4. ✅ Implement lazy loading for non-critical features

**Expected Impact**: 85% size reduction, minimal impact on customer Core Web Vitals

## Expected Results

### Before Optimization
- Bundle Size: 252KB
- Time to Interactive: 2-3 seconds
- SEO Impact: Moderate negative impact
- Core Web Vitals: Likely to trigger warnings

### After Optimization
- Bundle Size: ~35KB (86% reduction)
- Time to Interactive: 0.3-0.5 seconds
- SEO Impact: Minimal negative impact
- Core Web Vitals: Green scores maintained

### Key Metrics Improvement
- **Lighthouse Performance Score**: +25-35 points
- **First Contentful Paint**: -60% improvement
- **Time to Interactive**: -80% improvement
- **Cumulative Layout Shift**: Minimized through proper loading

## Risk Mitigation

### 1. Functionality Preservation
- Implement comprehensive testing suite
- Gradual rollout with feature flags
- Fallback mechanisms for critical features

### 2. Browser Compatibility
- Progressive enhancement approach
- Polyfills for older browsers
- Graceful degradation strategies

### 3. Development Velocity
- Maintain TypeScript for better DX
- Create component library for reusability
- Automated testing and deployment

## Conclusion

The current plugin has significant performance optimization opportunities. By implementing the recommended changes, we can reduce the bundle size by 85% while maintaining functionality and significantly improving the impact on customer landing pages' Core Web Vitals scores.

The key is to prioritize the high-impact changes first (async loading, dependency reduction) and then systematically work through the architectural improvements for maximum SEO benefit.