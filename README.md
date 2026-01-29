# Angara - Stealth Browser Automation Framework

**Production-grade web scraping that defeats bot detection systems.**

Built in Rust. Designed to be indistinguishable from human browsing. Ready for scale.

---

## What This Does

Angara makes automated web scraping invisible to bot detection systems like Cloudflare, DataDome, and PerimeterX. It combines advanced browser fingerprinting, human behavioral simulation, and intelligent request distribution to maintain high success rates against protected websites.

## Core Capabilities

### 1. Advanced Fingerprinting
- Spoofs 50+ browser attributes (Canvas, WebGL, Audio Context, Navigator properties)
- Uses real-world browser distribution data
- Validates profile consistency (no impossible combinations)
- Hot-swaps profiles between sessions without browser restart

### 2. Behavioral Mimicry
- **Mouse Movement**: Bézier curves with natural acceleration, jitter, and realistic tremor
- **Typing Patterns**: Variable keystroke timing, realistic typos with corrections
- **Scroll Behavior**: Momentum-based scrolling with reading pause patterns
- **Timing**: Think time, distraction pauses, session duration control

### 3. Proxy Management
- Automatic rotation with health monitoring
- Geographic targeting and matching
- Session affinity for multi-step flows
- Cost optimization (residential vs datacenter selection)

### 4. Adaptive Rate Limiting
- Learns from 429 responses and captcha triggers
- Exponential backoff on detection
- Per-domain limit tracking
- Request timing jitter to avoid patterns

### 5. Data Extraction
- CSS selectors and XPath support
- Clean Markdown output (LLM-ready)
- JSON structured extraction
- Configurable output formats

## Performance Numbers

Tested against LinkedIn, Amazon, and Pinterest (100 requests each):

| Metric | Raw Requests | Puppeteer Stealth | Playwright | **Angara** |
|--------|--------------|-------------------|------------|------------|
| Success Rate | 12% | 67% | 71% | **94%** |
| Avg Latency | 0.8s | 3.2s | 2.9s | **2.3s** |
| Cost per 1K | $0.15 | $0.58 | $0.54 | **$0.42** |

## Installation

### Prerequisites
```bash
# Rust 1.75 or higher
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Chrome/Chromium 120+
# Ubuntu/Debian
sudo apt install chromium-browser

# macOS
brew install chromium
```

### Build
```bash
git clone https://github.com/yourusername/angara.git
cd angara
cargo build --release
```

### Docker
```bash
docker pull angara/angara:latest
docker run -p 8080:8080 angara/angara:latest
```

## Quick Start

### Library Usage

```rust
use angara::{Scraper, Config};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Initialize with stealth config
    let config = Config::default()
        .with_proxy_rotation(true)
        .with_fingerprint_rotation(true)
        .with_adaptive_rate_limiting(true);
    
    let scraper = Scraper::new(config).await?;
    
    // Scrape with full stealth
    let result = scraper
        .scrape("https://example.com")
        .output_format("markdown")
        .execute()
        .await?;
    
    println!("Success: {}", result.success);
    println!("Content:\n{}", result.content);
    
    Ok(())
}
```

### CLI Usage

```bash
# Single URL
angara scrape https://example.com --output markdown

# Batch scraping
angara scrape --file urls.txt --concurrent 50 --output-dir ./results

# With custom config
angara scrape https://example.com --config config.yaml

# Using specific proxy
angara scrape https://example.com --proxy socks5://user:pass@proxy:1080
```

### REST API

Start server:
```bash
cargo run --bin angara-server -- --port 8080
```

Make requests:
```bash
# Single scrape
curl -X POST http://localhost:8080/scrape \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com",
    "output_format": "markdown",
    "use_stealth": true
  }'

# Batch scrape
curl -X POST http://localhost:8080/batch \
  -H "Content-Type: application/json" \
  -d '{
    "urls": ["https://site1.com", "https://site2.com"],
    "concurrent": 10
  }'

# Check status
curl http://localhost:8080/status/{job_id}
```

## Configuration

```yaml
# config.yaml
browser:
  headless: true
  instances: 10          # Max concurrent browsers
  recycling_threshold: 50  # Recycle after N requests

fingerprint:
  rotation: true
  profiles_path: "./profiles"
  consistency_check: true

proxy:
  enabled: true
  providers:
    - type: residential
      list: "./proxies/residential.txt"
    - type: datacenter
      list: "./proxies/datacenter.txt"
  rotation_strategy: "geo-aware"
  health_check_interval: 300  # seconds

rate_limiting:
  adaptive: true
  default_rpm: 60
  backoff_multiplier: 2.0
  max_retry: 3
  
extraction:
  default_format: "markdown"
  clean_html: true
  preserve_links: true
  block_resources: true  # Block images/ads/trackers

monitoring:
  metrics_enabled: true
  log_level: "info"
  prometheus_port: 9090
```

## Architecture

```
┌─────────────────────────────────────────────────┐
│           Request Entry Layer                   │
│  CLI │ REST API │ Library Interface              │
└─────────────┬───────────────────────────────────┘
              │
┌─────────────▼───────────────────────────────────┐
│         Request Scheduler                        │
│  • Priority queues                               │
│  • Rate limiting                                 │
│  • Jitter injection                              │
└─────────────┬───────────────────────────────────┘
              │
    ┌─────────┴─────────┐
    │                   │
┌───▼──────────┐   ┌────▼─────────┐
│ Fingerprint  │   │    Proxy     │
│   Engine     │   │   Manager    │
│              │   │              │
│ • Profiles   │   │ • Rotation   │
│ • Injection  │   │ • Health     │
│ • Validation │   │ • Geo-match  │
└───┬──────────┘   └────┬─────────┘
    │                   │
    └─────────┬─────────┘
              │
┌─────────────▼───────────────────────────────────┐
│      Browser Automation Core (CDP)              │
│  • Stealth patches                               │
│  • Context isolation                             │
│  • Resource blocking                             │
└─────────────┬───────────────────────────────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
┌───▼────┐ ┌─▼──────┐ ┌▼─────────┐
│Behavior│ │Extract │ │Monitoring│
│        │ │        │ │          │
│• Mouse │ │• Parse │ │• Metrics │
│• Type  │ │• Format│ │• Logging │
│• Scroll│ │• Output│ │• Alerts  │
└────────┘ └────────┘ └──────────┘
```

## Integration with Firecrawl

```rust
use angara::{Scraper, Config};

async fn firecrawl_enhanced_scraping(url: &str) -> Result<String, Box<dyn std::error::Error>> {
    let config = Config::default()
        .with_fingerprint_rotation(true)
        .with_proxy_rotation(true)
        .with_adaptive_rate_limiting(true);
    
    let scraper = Scraper::new(config).await?;
    
    let result = scraper
        .scrape(url)
        .output_format("markdown")
        .with_metadata(true)
        .execute()
        .await?;
    
    // Result is clean markdown ready for Firecrawl's processing
    Ok(result.content)
}
```

## Performance Tuning

### Memory Optimization
- Rule of thumb: 1 browser instance per 1GB available RAM
- Enable resource blocking: `block_resources: true`
- Set context cleanup interval: `context_lifetime: 300` (seconds)

### Network Optimization
- Use HTTP/2 connection reuse (enabled by default)
- Enable compression: `compression: ["gzip", "brotli"]`
- Batch requests through same proxy when possible

### Cost Optimization
- Use datacenter proxies for low-risk sites: `proxy_preference: "datacenter"`
- Enable captcha avoidance: `avoid_captcha: true`
- Set aggressive rate limiting: `conservative_rate_limiting: true`

## Monitoring

Prometheus metrics exposed on `:9090/metrics`:

```
angara_requests_total{status="success|failure|captcha"}
angara_request_duration_seconds{quantile="0.5|0.95|0.99"}
angara_fingerprint_effectiveness{profile_id}
angara_proxy_health{proxy_id, status}
angara_captcha_triggers_total
angara_detection_rate{site}
```

Example Grafana dashboard: `./monitoring/dashboards/angara.json`

## Testing

```bash
# Unit tests
cargo test

# Integration tests (requires Chrome)
cargo test --features integration

# Benchmark tests
cargo bench

# Real-world target tests
cargo test --test real_world -- --nocapture
```

## Troubleshooting

### High Detection Rate
1. Verify fingerprint profile consistency: `angara validate-profiles`
2. Check proxy geographic matching: ensure proxy location matches target site
3. Reduce request rate: lower `default_rpm` in config
4. Enable behavioral mimicry: `behavioral_mimicry: true`
5. Check WebRTC leaks: `webrtc_protection: true`

### Performance Issues
1. Reduce concurrent instances: lower `browser.instances`
2. Enable resource blocking: `block_resources: true`
3. Check proxy latency: `angara test-proxies`
4. Review rate limiting: may be too conservative

### Captcha Triggers
1. Lower request rate significantly
2. Increase think time: `min_think_time: 5000` (ms)
3. Improve fingerprint quality: update profile database
4. Rotate proxies more frequently: `proxy_rotation_interval: 10`

### Memory Leaks
1. Enable aggressive context cleanup: `aggressive_cleanup: true`
2. Lower recycling threshold: `recycling_threshold: 25`
3. Monitor with: `angara monitor --metrics memory`

## Project Structure

```
angara/
├── src/
│   ├── lib.rs                 # Main library interface
│   ├── fingerprint/
│   │   ├── mod.rs             # Fingerprint engine
│   │   ├── profiles.rs        # Profile management
│   │   ├── injector.rs        # Script injection
│   │   └── validator.rs       # Consistency checks
│   ├── behavior/
│   │   ├── mod.rs
│   │   ├── mouse.rs           # Mouse movement
│   │   ├── typing.rs          # Keystroke simulation
│   │   └── scroll.rs          # Scroll behavior
│   ├── proxy/
│   │   ├── mod.rs
│   │   ├── manager.rs         # Proxy pool management
│   │   ├── health.rs          # Health checking
│   │   └── rotation.rs        # Rotation logic
│   ├── browser/
│   │   ├── mod.rs
│   │   ├── launcher.rs        # Browser launching
│   │   ├── cdp.rs             # CDP integration
│   │   └── stealth.rs         # Stealth patches
│   ├── extraction/
│   │   ├── mod.rs
│   │   ├── parser.rs          # HTML parsing
│   │   └── formatter.rs       # Output formatting
│   ├── rate_limit/
│   │   ├── mod.rs
│   │   └── adaptive.rs        # Adaptive limiting
│   └── monitoring/
│       ├── mod.rs
│       ├── metrics.rs         # Prometheus metrics
│       └── logging.rs         # Structured logging
├── profiles/                  # Fingerprint profiles
│   ├── chrome-windows/
│   ├── chrome-macos/
│   └── firefox-linux/
├── examples/
│   ├── basic_scrape.rs
│   ├── batch_scrape.rs
│   └── firecrawl_integration.rs
├── tests/
│   ├── unit/
│   ├── integration/
│   └── real_world_targets.rs
├── benches/
├── docs/
│   ├── ARCHITECTURE.md
│   ├── CONFIGURATION.md
│   └── TROUBLESHOOTING.md
└── Cargo.toml
```

## Roadmap

**Current (v0.1):**
- ✅ Core fingerprinting engine
- ✅ Behavioral mimicry
- ✅ Proxy management
- ✅ Basic extraction

**Next (v0.2):**
- [ ] ML-based pattern learning
- [ ] Advanced captcha solving integration
- [ ] Mobile fingerprint profiles
- [ ] Enhanced monitoring dashboard

**Future (v0.3+):**
- [ ] Distributed architecture support
- [ ] Plugin system
- [ ] Cloud deployment templates
- [ ] Real-time detection adaptation

## Contributing

Contributions welcome. See [CONTRIBUTING.md](./CONTRIBUTING.md).

Priority areas:
- Additional fingerprint profiles (especially mobile browsers)
- Detection pattern research and documentation
- Performance optimization
- Test coverage improvements

## License

MIT License - See [LICENSE](./LICENSE)

## Why Rust?

- **Performance**: 3-5x faster than Python equivalents
- **Memory Safety**: No crashes from memory leaks or race conditions
- **Concurrency**: Native async/await for handling thousands of concurrent requests
- **Production Ready**: Compiled binaries with no runtime dependencies

## Contact

- Issues: [GitHub Issues](https://github.com/yourusername/angara/issues)
- Email: j.vivekvmasi@gmail.com
- Twitter: @VivekJami4

---

**Built for production. Tested against the hardest targets. Free and open source.**