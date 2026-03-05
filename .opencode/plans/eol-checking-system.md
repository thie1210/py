# Implementation Plan: End-of-Life Checking System

**Version**: 1.0  
**Date**: 2026-03-05  
**Status**: Ready for Implementation  
**Target Release**: v0.3.0 (Phase 1), v0.3.1 (Phase 2), v0.3.2 (Phase 3)

---

## Overview

**Goal**: Implement comprehensive End-of-Life (EOL) checking for Python and all installed packages using endoflife.date API with intelligent caching and graceful degradation.

**Philosophy**: 
- Show by default (no need for special flags)
- Warn 8 months before EOL
- Graceful degradation when API unavailable
- All installed packages checked systematically

---

## Design Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| API Source | endoflife.date | Free, open, comprehensive (434 products) |
| Cache location | `~/.local/share/srpt/cache/eol_*.json` | Prefixed files in existing cache dir |
| Cache TTL | 24 hours | EOL dates don't change often |
| Cache age warning | 7 days | Warn if cache is stale |
| API rate limit handling | 100ms delay between requests | Conservative, no published limits |
| Offline behavior | Show cached data with warning | Graceful degradation |
| Unknown packages | Skip silently | Not all packages in database |
| Warning threshold | 8 months before EOL | User preference |
| Implementation | Option C (Hybrid) | Best of both worlds |

---

## API Details

### endoflife.date API

**Base URL**: `https://endoflife.date/api`

**Endpoints**:
- `GET /api/v1/products/{product}.json` - Get EOL data for a product
- `GET /api/v1/products.json` - List all products

**Example Response** (Python):
```json
[
  {
    "cycle": "3.14",
    "releaseDate": "2025-10-07",
    "eol": "2030-10-31",
    "latest": "3.14.3",
    "latestReleaseDate": "2026-02-03",
    "pep": "PEP-0745",
    "lts": false,
    "support": "2027-10-01"
  },
  ...
]
```

**Example Response** (Django):
```json
[
  {
    "cycle": "6.0",
    "releaseDate": "2025-12-03",
    "eol": "2027-04-30",
    "supportedPythonVersions": "3.12 - 3.14",
    "latest": "6.0.3",
    "latestReleaseDate": "2026-03-03",
    "lts": false,
    "support": "2026-08-31"
  },
  ...
]
```

**Rate Limits**: Not published. Hosted on Netlify free tier.
**Recommendation**: Use 100ms delay between requests, implement caching.

---

## Architecture

### Module Structure

```
src/srpt/
├── eol.py                    # NEW: EOL checking module
│   ├── EOLChecker class
│   ├── fetch_eol_data()
│   ├── get_cached_eol()
│   ├── set_cached_eol()
│   ├── check_package_eol()
│   ├── check_python_eol()
│   └── batch_check_packages()
├── health.py                 # UPDATED: Add EOL section
└── status.py                 # UPDATED: Add EOL summary
```

### Cache Structure

**Location**: `~/.local/share/srpt/cache/`

**Files**:
```
eol_python.json
eol_django.json
eol_numpy.json
eol_nodejs.json
...
```

**Format**:
```json
{
  "product": "python",
  "data": [
    {"cycle": "3.14", "eol": "2030-10-31", ...},
    ...
  ],
  "cached_at": "2026-03-05T01:50:00Z",
  "expires_at": "2026-03-06T01:50:00Z",
  "source": "https://endoflife.date/api/python.json"
}
```

---

## EOL Data Structure

### For Each Package

```python
{
  "name": "django",
  "installed_version": "6.0.2",
  "cycle": "6.0",
  "eol_date": "2027-04-30",
  "support_date": "2026-08-31",
  "is_eol": False,
  "is_approaching_eol": False,  # Within 8 months
  "days_until_eol": 415,
  "latest_in_cycle": "6.0.3",
  "supported_python_versions": "3.12 - 3.14",
  "lts": False,
  "status": "✓ Supported",
  "warning": None,
  "api_available": True,
  "cache_age_hours": 2
}
```

### Status Values

- `✓ Supported` - No action needed
- `! Approaching EOL` - Plan upgrade (within 8 months)
- `✗ End of Life` - Upgrade recommended
- `○ Not tracked` - EOL data unavailable
- `⚠ Cached data` - API unavailable, using cache

---

## Implementation Phases

### Phase 1: Core EOL Module (v0.3.0)

**Duration**: 3-4 days  
**Priority**: High - Foundation for EOL checking

#### Day 1-2: EOL Module Core

**Files to create**:
- `src/srpt/eol.py`

**Key classes and functions**:

```python
class EOLChecker:
    """Main EOL checking class."""
    
    def __init__(self, cache_dir: Path = None):
        """Initialize with cache directory."""
        
    async def fetch_eol_data(self, product: str) -> List[Dict]:
        """
        Fetch EOL data from endoflife.date API.
        
        Args:
            product: Product name (e.g., "python", "django")
        
        Returns:
            List of EOL cycle data
        
        Raises:
            httpx.HTTPError: If API request fails
        """
    
    def get_cached_eol(self, product: str) -> Optional[Dict]:
        """
        Get cached EOL data if not expired.
        
        Args:
            product: Product name
        
        Returns:
            Cached data or None if expired/missing
        """
    
    def set_cached_eol(self, product: str, data: List[Dict]) -> None:
        """
        Cache EOL data with 24-hour TTL.
        
        Args:
            product: Product name
            data: EOL data to cache
        """
    
    def check_package_eol(
        self,
        name: str,
        version: str,
        eol_data: List[Dict]
    ) -> Dict:
        """
        Check EOL status for a specific package version.
        
        Args:
            name: Package name
            version: Installed version
            eol_data: EOL data for the product
        
        Returns:
            EOL status dict
        """
    
    async def check_python_eol(self, version: str) -> Dict:
        """
        Check EOL status for Python version.
        
        Args:
            version: Python version (e.g., "3.13.12")
        
        Returns:
            EOL status dict
        """
    
    async def batch_check_packages(
        self,
        packages: List[Tuple[str, str]]
    ) -> List[Dict]:
        """
        Check EOL status for multiple packages in parallel.
        
        Args:
            packages: List of (name, version) tuples
        
        Returns:
            List of EOL status dicts
        """
```

**Helper functions**:

```python
def normalize_package_name(pypi_name: str) -> str:
    """
    Convert PyPI package name to endoflife.date product name.
    
    Examples:
        "python-dateutil" → "dateutil"
        "Pillow" → "pillow"
        "django" → "django"
    """

def find_eol_cycle(version: str, eol_data: List[Dict]) -> Optional[Dict]:
    """
    Find matching EOL cycle for installed version.
    
    Examples:
        "6.0.2" → matches cycle "6.0"
        "3.14.3" → matches cycle "3.14"
    """

def calculate_eol_status(
    eol_date: str,
    support_date: Optional[str] = None
) -> Tuple[str, bool, bool, int]:
    """
    Calculate EOL status from dates.
    
    Returns:
        (status, is_eol, is_approaching_eol, days_until_eol)
    """
```

**Package name mapping**:

```python
EOL_NAME_MAPPING = {
    "python-dateutil": "dateutil",
    "pyyaml": "yaml",
    "pillow": "pillow",
    "psycopg2-binary": "postgresql",  # Maps to PostgreSQL driver
    "psycopg2": "postgresql",
    # Add more as discovered
}
```

#### Day 3: Integration with Health Check

**Files to update**:
- `src/srpt/health.py`

**New function**:

```python
async def check_eol_status(project_root: Path) -> Dict:
    """
    Check EOL status for Python and all installed packages.
    
    Returns:
        Dict with EOL information:
        - python: Python EOL status
        - packages: List of package EOL statuses
        - api_available: Whether API was reachable
        - cache_age_hours: Age of cache if used
        - warnings: Count of EOL/approaching EOL packages
    """
```

**Enhanced `format_health_report()`**:

Add new section after COMPATIBILITY:

```
END OF LIFE
  ✓ Python 3.13: Supported (EOL: 2029-10-31)
  ✓ Django 6.0: Supported (EOL: 2027-04-30)
  ! NumPy 1.26: Approaching EOL (2025-09-17, 194 days)
    → Upgrade to NumPy 2.4+
  ✗ Requests 2.28: End of Life (2024-01-15)
    → Upgrade to Requests 2.32+
  
  API Status: ✓ Connected
  Cache: Updated 2 hours ago
```

**Offline mode**:

```
END OF LIFE
  ⚠ API unavailable - showing cached data (2 days old)
  ✓ Python 3.13: Supported (cached data)
  ...
```

#### Day 4: Testing

**Tests to create**:
- `tests/test_eol.py`

**Test cases**:

```python
- test_fetch_eol_data_success()
- test_fetch_eol_data_404()
- test_fetch_eol_data_timeout()
- test_cache_write_read()
- test_cache_expiration()
- test_version_matching()
- test_package_name_normalization()
- test_eol_status_calculation()
- test_approaching_eol_detection()
- test_offline_mode_with_cache()
- test_batch_checking()
```

---

### Phase 2: Package EOL Checking (v0.3.1)

**Duration**: 2-3 days  
**Priority**: Medium - Complete EOL coverage

#### Day 5-6: Batch Checking & Progress

**Enhancements**:

1. **Parallel API fetching**:
```python
async def batch_fetch_eol_data(products: List[str]) -> Dict[str, List[Dict]]:
    """Fetch EOL data for multiple products in parallel."""
    async with httpx.AsyncClient(http2=True, timeout=10.0) as client:
        tasks = [fetch_eol_for_product(client, product) for product in products]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        return {product: result for product, result in zip(products, results)}
```

2. **Progress indicators**:
```
Checking EOL status...
  Fetching Python... ✓
  Fetching Django... ✓
  Fetching NumPy... ✓
  ...
  Checked 50 packages (3.2s)
```

3. **Smart caching**:
- Check cache first
- Only fetch missing/expired products
- Batch fetch in parallel

#### Day 7: Status Command Integration

**Files to update**:
- `src/srpt/status.py`

**Add to HEALTH section**:

```
HEALTH
  ✓ Vulnerabilities: 0 found
  ! Outdated: 2 packages
  ! EOL: 1 package approaching, 0 end of life
    → Run 'srpt health' for details
```

---

### Phase 3: Polish & Configuration (v0.3.2)

**Duration**: 2 days  
**Priority**: Low - Enhanced features

#### Day 8: Configuration & Flags

**New flags**:

```bash
srpt health --refresh-eol    # Force refresh EOL cache
srpt health --no-eol         # Skip EOL checking
```

**Configuration** (future - not in v0.3.2):

```toml
[eol]
enabled = true
cache_ttl_hours = 24
warn_months_before_eol = 8
api_timeout_seconds = 10
api_base_url = "https://endoflife.date/api"
```

#### Day 9: Error Handling & Edge Cases

**Edge cases to handle**:

1. **API unavailable**:
   - Show cached data with warning
   - Indicate cache age
   - Suggest `--refresh-eol` when online

2. **Product not in database**:
   - Skip silently
   - Don't count in warnings
   - Log for debugging

3. **Version not matching any cycle**:
   - Show as "unknown version"
   - Suggest checking package

4. **Cache corrupted**:
   - Delete and re-fetch
   - Log warning

5. **Rate limiting (429)**:
   - Back off and retry
   - Use cached data if available

---

## User Experience

### Example Output

#### Normal Operation

```
$ srpt health

SRPT HEALTH CHECK
  ✓ srpt version: 0.3.0 (latest: 0.3.0)
  ✓ Python: 3.13.12
  ✓ Cache: 72.5 MB

SECURITY
  ✓ Vulnerabilities: 0 found

DEPENDENCIES
  Installed: 50
  ! Outdated: 2
    • django 6.0.2 → 6.0.3
    • requests 2.32.0 → 2.32.5
  → Run 'srpt update' to update all

COMPATIBILITY
  Python 3.12: ✓ compatible
  Python 3.13: ✓ compatible

END OF LIFE
  ✓ Python 3.13: Supported (EOL: 2029-10-31)
  ✓ Django 6.0: Supported (EOL: 2027-04-30)
  ! NumPy 1.26: Approaching EOL (194 days)
    → Upgrade to: NumPy 2.4.2
  ✗ Requests 2.28: End of Life (expired 2024-01-15)
    → Upgrade to: Requests 2.32.5
  
  API Status: ✓ Connected
  Cache: Updated 2 hours ago

SUMMARY
  ! 3 warnings
  → Run 'srpt health --full' for all 50 packages
```

#### Offline Mode

```
END OF LIFE
  ⚠ API unavailable - showing cached data (2 days old)
  ✓ Python 3.13: Supported (cached)
  ✓ Django 6.0: Supported (cached)
  ...
  
  → Run 'srpt health --refresh-eol' when online to update
```

#### Approaching EOL

```
END OF LIFE
  ! Python 3.10: Approaching EOL (2026-10-31, 240 days)
    → Upgrade to: Python 3.14.3 (latest)
    → Or: Python 3.13.12 (stable)
  ! Django 4.2: Approaching EOL (2026-04-30, 56 days)
    → Upgrade to: Django 5.2.12 (LTS until 2028-04-30)
```

---

## Performance Considerations

### Expected Performance

| Scenario | Time | Notes |
|----------|------|-------|
| First run (no cache) | 3-5s | 50 packages, parallel fetch |
| Cached run | <1s | All data from cache |
| Offline with cache | <1s | Cache read only |
| Offline no cache | <1s | Skip EOL section |

### Optimizations

1. **HTTP/2 multiplexing** - Multiple requests over single connection
2. **Parallel fetching** - asyncio.gather() for concurrent requests
3. **Smart caching** - Only fetch missing/expired products
4. **100ms delay** - Conservative rate limit handling
5. **Lazy loading** - Only fetch what's needed

---

## Testing Strategy

### Unit Tests

**File**: `tests/test_eol.py`

```python
class TestEOLChecker:
    def test_normalize_package_name()
    def test_find_eol_cycle()
    def test_calculate_eol_status()
    def test_cache_write_read()
    def test_cache_expiration()
    def test_fetch_eol_data_success()
    def test_fetch_eol_data_404()
    def test_fetch_eol_data_timeout()
    def test_batch_check_packages()
    def test_offline_mode()

class TestEOLIntegration:
    def test_health_with_eol()
    def test_status_with_eol_summary()
    def test_offline_graceful_degradation()
```

### Integration Tests

```python
- test_full_eol_workflow()
- test_parallel_fetching()
- test_cache_invalidation()
- test_api_failure_handling()
```

### Mock Data

```python
MOCK_PYTHON_EOL = [
    {"cycle": "3.14", "eol": "2030-10-31", ...},
    {"cycle": "3.13", "eol": "2029-10-31", ...},
    ...
]

MOCK_DJANGO_EOL = [
    {"cycle": "6.0", "eol": "2027-04-30", ...},
    ...
]
```

---

## Documentation Updates

### Files to Create

**`docs/EOL_CHECKING.md`**:
- What is EOL checking?
- How it works
- Understanding EOL status
- Offline behavior
- Configuration
- API details

### Files to Update

**`README.md`**:
- Add EOL checking to features
- Update health command examples

**`CHANGELOG.md`**:
- Add v0.3.0 release notes

---

## Known Products in endoflife.date

**Python packages tracked** (as of 2026-03-05):
- ✅ Python
- ✅ Django
- ✅ NumPy
- ✅ Node.js (if used in project)
- ✅ React (if used in project)
- ✅ Many frameworks and tools

**Not tracked**:
- Most PyPI packages
- Internal/proprietary packages
- Very new or obscure packages

**Strategy**: Check all installed packages, skip those not in database.

---

## Success Criteria

### Phase 1 Complete When:
- [ ] EOL module fetches data from endoflife.date
- [ ] Caching works correctly (24hr TTL)
- [ ] Python EOL status shown in health check
- [ ] Offline mode shows cached data
- [ ] All unit tests pass

### Phase 2 Complete When:
- [ ] All installed packages checked for EOL
- [ ] Parallel fetching works
- [ ] Progress indicators shown
- [ ] Status command shows EOL summary
- [ ] Integration tests pass

### Phase 3 Complete When:
- [ ] `--refresh-eol` flag works
- [ ] All edge cases handled
- [ ] Error messages are clear
- [ ] Documentation complete
- [ ] Ready for v0.3.0 release

---

## Open Questions

1. **Cache cleanup**: Should we auto-delete old cache files?
   - Recommendation: Yes, delete files > 7 days old

2. **API timeout**: What's acceptable?
   - Recommendation: 10 seconds per request

3. **Parallel requests**: How many concurrent?
   - Recommendation: 10 concurrent requests

4. **Cache size limit**: Should we limit total cache size?
   - Recommendation: No, EOL data is small (~50KB for 50 packages)

---

## Next Steps

**Begin Phase 1 implementation**:
1. Create `src/srpt/eol.py` with core functionality
2. Implement caching system
3. Add Python EOL checking
4. Integrate with health command
5. Write tests

---

**Plan Status**: ✅ Ready for Implementation
