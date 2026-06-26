# Lemon Squeezy Licensing Implementation Guide

This document explains how Lemon Squeezy licensing can be implemented in a desktop application. Use this as a template for adding licensing to your own applications.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [File Structure](#file-structure)
3. [Lemon Squeezy Setup](#lemon-squeezy-setup)
4. [Python Client Implementation](#python-client-implementation)
5. [Cloudflare Worker (Server-Side)](#cloudflare-worker-server-side)
6. [UI Integration](#ui-integration)
7. [PyInstaller Configuration](#pyinstaller-configuration)
8. [Security Considerations](#security-considerations)

---

## Architecture Overview

The licensing system has two modes:

### Mode 1: Direct Validation (Subscriptions)
```
App --> Lemon Squeezy API --> License Valid/Invalid
```
- Best for subscription products where credits aren't tracked
- Simpler setup, no server infrastructure needed
- License validates directly with Lemon Squeezy's public endpoints

### Mode 2: Cloudflare Worker (Credit Packs)
```
App --> Cloudflare Worker --> Lemon Squeezy API
                |
                v
         Cloudflare KV (credits storage)
```
- Server-side credit tracking prevents local tampering
- Webhooks automatically add credits on purchase
- HMAC-SHA256 signed requests for security

---

## File Structure

```
your_app/
|-- utils/
|   |-- licensing/
|       |-- __init__.py          # Module exports
|       |-- lemon_squeezy.py     # Low-level API client
|       |-- license_manager.py   # High-level license management
|       |-- usage_client.py      # Cloudflare Worker client
|-- server/
|   |-- cloudflare_worker.js     # Cloudflare Worker code
|   |-- CLOUDFLARE_SETUP.md      # Setup instructions
|-- your_app.py                  # Main application with UI integration
```

---

## Lemon Squeezy Setup

### 1. Create Your Product

1. Create a Lemon Squeezy account at https://lemonsqueezy.com
2. Create a Store
3. Create a Product with variants:
   - **Credit Packs** (one-time): "100 Credits", "500 Credits", "1000 Credits"
   - **Subscriptions**: "Weekly Subscription", "Daily Subscription"
4. Enable "License keys" for the product under Product Settings

### 2. Get Your IDs

- **Store ID**: Found in URL `https://app.lemonsqueezy.com/stores/XXXXX`
- **Product ID**: Found in URL `https://app.lemonsqueezy.com/products/XXXXX`
- **API Key**: Settings > API > Create new key

### 3. Configure Webhooks (for Credit Packs)

1. Go to Settings > Webhooks
2. Add webhook URL: `https://your-worker.workers.dev/webhook`
3. Select events: `order_created`, `subscription_created`, `subscription_updated`
4. Save the signing secret for webhook verification

---

## Python Client Implementation

### 1. `__init__.py` - Module Exports

```python
"""
Licensing module - Lemon Squeezy integration.
"""
from .license_manager import LicenseManager
from .lemon_squeezy import LicenseInfo, LemonSqueezyClient

__all__ = ['LicenseManager', 'LicenseInfo', 'LemonSqueezyClient']
```

### 2. `lemon_squeezy.py` - Low-Level API Client

This handles direct communication with Lemon Squeezy's API.

```python
"""
Lemon Squeezy API client for license validation.
"""
import requests
from dataclasses import dataclass
from typing import Optional, Dict, Any
import logging

logger = logging.getLogger(__name__)

LEMON_SQUEEZY_API = "https://api.lemonsqueezy.com/v1"
LICENSE_VALIDATE_URL = f"{LEMON_SQUEEZY_API}/licenses/validate"
LICENSE_ACTIVATE_URL = f"{LEMON_SQUEEZY_API}/licenses/activate"
LICENSE_DEACTIVATE_URL = f"{LEMON_SQUEEZY_API}/licenses/deactivate"


@dataclass
class LicenseInfo:
    """Information about a validated license."""
    valid: bool
    license_key: str = ""
    license_key_id: Optional[int] = None
    status: str = ""  # inactive, active, expired, disabled
    activation_limit: int = 0
    activation_usage: int = 0

    # Subscription info
    subscription_id: Optional[int] = None
    subscription_item_id: Optional[int] = None

    # Customer/order info
    store_id: Optional[int] = None
    order_id: Optional[int] = None
    product_id: Optional[int] = None
    product_name: str = ""
    variant_id: Optional[int] = None
    variant_name: str = ""
    customer_id: Optional[int] = None
    customer_name: str = ""
    customer_email: str = ""

    # Usage tracking
    credits_used: int = 0
    credits_limit: int = 0

    # Instance info (for device tracking)
    instance_id: Optional[str] = None
    instance_name: str = ""

    # Subscription expiration
    expires_at: Optional[str] = None

    # Error info
    error_message: Optional[str] = None

    @property
    def credits_remaining(self) -> int:
        if self.credits_limit == 0:
            return 999999  # Unlimited
        return max(0, self.credits_limit - self.credits_used)

    @property
    def is_active(self) -> bool:
        return self.valid and self.status == "active"

    @property
    def days_remaining(self) -> Optional[int]:
        """Calculate days remaining until subscription expires."""
        if not self.expires_at:
            return None
        try:
            from datetime import datetime, timezone
            expires = datetime.fromisoformat(self.expires_at.replace('Z', '+00:00'))
            now = datetime.now(timezone.utc)
            delta = expires - now
            return max(0, delta.days)
        except Exception:
            return None


class LemonSqueezyClient:
    """Client for Lemon Squeezy API."""

    def __init__(self, api_key: str = ""):
        """
        Initialize the client.
        Note: API key not needed for validate/activate endpoints.
        """
        self.api_key = api_key
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Accept": "application/vnd.api+json",
            "Content-Type": "application/vnd.api+json"
        }
        self.form_headers = {"Accept": "application/json"}

    def validate_license(self, license_key: str) -> LicenseInfo:
        """Validate a license key without activating."""
        try:
            response = requests.post(
                LICENSE_VALIDATE_URL,
                data={"license_key": license_key},
                headers=self.form_headers,
                timeout=30
            )
            if response.status_code == 200:
                return self._parse_license_response(response.json(), license_key)
            return LicenseInfo(valid=False, license_key=license_key,
                             error_message=f"HTTP {response.status_code}")
        except requests.exceptions.Timeout:
            return LicenseInfo(valid=False, license_key=license_key,
                             error_message="Connection timeout")
        except Exception as e:
            return LicenseInfo(valid=False, license_key=license_key,
                             error_message=str(e))

    def activate_license(self, license_key: str, instance_name: str = "MyApp") -> LicenseInfo:
        """Activate a license key for this device."""
        try:
            response = requests.post(
                LICENSE_ACTIVATE_URL,
                data={"license_key": license_key, "instance_name": instance_name},
                headers=self.form_headers,
                timeout=30
            )
            if response.status_code == 200:
                data = response.json()
                info = self._parse_license_response(data, license_key)
                if "instance" in data and data["instance"]:
                    info.instance_id = data["instance"].get("id")
                    info.instance_name = data["instance"].get("name", instance_name)
                return info
            elif response.status_code == 400:
                data = response.json()
                return LicenseInfo(valid=False, license_key=license_key,
                                 error_message=data.get("error", "Activation limit reached"))
            return LicenseInfo(valid=False, license_key=license_key,
                             error_message=f"HTTP {response.status_code}")
        except Exception as e:
            return LicenseInfo(valid=False, license_key=license_key,
                             error_message=str(e))

    def deactivate_license(self, license_key: str, instance_id: str) -> bool:
        """Deactivate a license instance (logout)."""
        try:
            response = requests.post(
                LICENSE_DEACTIVATE_URL,
                data={"license_key": license_key, "instance_id": instance_id},
                headers=self.form_headers,
                timeout=30
            )
            return response.status_code == 200
        except Exception:
            return False

    def _parse_license_response(self, data: Dict[str, Any], license_key: str) -> LicenseInfo:
        """Parse the license validation/activation response."""
        valid = data.get("valid", False) or data.get("activated", False)
        license_data = data.get("license_key", {})
        meta = data.get("meta", {})

        return LicenseInfo(
            valid=valid,
            license_key=license_key,
            license_key_id=license_data.get("id"),
            status=license_data.get("status", ""),
            activation_limit=license_data.get("activation_limit", 0),
            activation_usage=license_data.get("activation_usage", 0),
            store_id=meta.get("store_id"),
            order_id=meta.get("order_id"),
            product_id=meta.get("product_id"),
            product_name=meta.get("product_name", ""),
            variant_id=meta.get("variant_id"),
            variant_name=meta.get("variant_name", ""),
            customer_id=meta.get("customer_id"),
            customer_name=meta.get("customer_name", ""),
            customer_email=meta.get("customer_email", ""),
            expires_at=license_data.get("expires_at"),
            error_message=data.get("error")
        )
```

### 3. `usage_client.py` - Cloudflare Worker Client

For server-side credit tracking:

```python
"""
Secure Credit Client - communicates with Cloudflare Worker.
"""
import os
import time
import hmac
import hashlib
import base64
import logging
from typing import Optional, Tuple
from dataclasses import dataclass
import requests

logger = logging.getLogger(__name__)

# Configure these for your deployment
WORKER_URL = os.environ.get("MYAPP_WORKER_URL", "https://myapp-usage.workers.dev")
SIGNING_SECRET = os.environ.get("MYAPP_SIGNING_SECRET", "your-secret-here")


@dataclass
class CreditInfo:
    """Information about current credits."""
    credits_total: int = 0
    credits_used: int = 0
    credits_remaining: int = 0
    is_subscription: bool = False
    error_message: Optional[str] = None

    @property
    def has_credits(self) -> bool:
        if self.is_subscription:
            return True
        return self.credits_remaining > 0

    def can_process(self, amount_needed: int) -> bool:
        if self.is_subscription:
            return True
        return self.credits_remaining >= amount_needed


@dataclass
class ActivationResult:
    """Result of license activation."""
    valid: bool = False
    activated: bool = False
    instance_id: Optional[str] = None
    license_key: Optional[str] = None
    customer_email: str = ""
    customer_name: str = ""
    product_name: str = ""
    variant_name: str = ""
    activation_limit: int = 0
    activation_usage: int = 0
    credits: Optional[CreditInfo] = None
    error_message: Optional[str] = None


class SecureCreditClient:
    """Client for secure credit management via Cloudflare Worker."""

    def __init__(self, worker_url: str = None, signing_secret: str = None):
        self.worker_url = worker_url or WORKER_URL
        self.signing_secret = signing_secret or SIGNING_SECRET
        self._license_key: Optional[str] = None

    def _compute_signature(self, message: str) -> str:
        """Compute HMAC-SHA256 signature."""
        signature = hmac.new(
            self.signing_secret.encode(),
            message.encode(),
            hashlib.sha256
        ).digest()
        return base64.b64encode(signature).decode()

    def _make_request(self, action: str, data: dict) -> dict:
        """Make a signed request to the worker."""
        data["action"] = action
        try:
            response = requests.post(self.worker_url, json=data, timeout=30)
            return response.json()
        except requests.exceptions.Timeout:
            return {"error": "Connection timeout"}
        except Exception as e:
            return {"error": str(e)}

    def activate(self, license_key: str, instance_name: str = "MyApp") -> ActivationResult:
        """Activate a license and get initial credits."""
        timestamp = int(time.time() * 1000)
        message = f"activate:{license_key}:{timestamp}"
        signature = self._compute_signature(message)

        result = self._make_request("activate", {
            "license_key": license_key,
            "instance_name": instance_name,
            "timestamp": timestamp,
            "signature": signature,
        })

        if "error" in result and not result.get("valid", False):
            return ActivationResult(valid=False, error_message=result.get("error"))

        if not result.get("valid", False):
            return ActivationResult(valid=False, error_message=result.get("error", "Invalid"))

        self._license_key = license_key

        credits = CreditInfo(
            credits_total=result.get("credits_total", 0),
            credits_used=result.get("credits_used", 0),
            credits_remaining=result.get("credits_remaining", 0),
            is_subscription=result.get("is_subscription", False),
        )

        return ActivationResult(
            valid=True,
            activated=result.get("activated", False),
            instance_id=result.get("instance_id"),
            license_key=license_key,
            customer_email=result.get("customer_email", ""),
            customer_name=result.get("customer_name", ""),
            product_name=result.get("product_name", ""),
            variant_name=result.get("variant_name", ""),
            activation_limit=result.get("activation_limit", 0),
            activation_usage=result.get("activation_usage", 0),
            credits=credits,
        )

    def get_credits(self, license_key: str = None) -> CreditInfo:
        """Get current credit balance from server."""
        license_key = license_key or self._license_key
        if not license_key:
            return CreditInfo(error_message="No license key")

        timestamp = int(time.time() * 1000)
        message = f"get_credits:{license_key}:{timestamp}"
        signature = self._compute_signature(message)

        result = self._make_request("get_credits", {
            "license_key": license_key,
            "timestamp": timestamp,
            "signature": signature,
        })

        if "error" in result:
            return CreditInfo(error_message=result.get("error"))

        return CreditInfo(
            credits_total=result.get("credits_total", 0),
            credits_used=result.get("credits_used", 0),
            credits_remaining=result.get("credits_remaining", 0),
            is_subscription=result.get("is_subscription", False),
        )

    def check_credits(self, amount_needed: int, license_key: str = None) -> Tuple[bool, CreditInfo]:
        """Check if user has enough credits."""
        license_key = license_key or self._license_key
        if not license_key:
            return False, CreditInfo(error_message="No license key")

        timestamp = int(time.time() * 1000)
        message = f"check_credits:{license_key}:{amount_needed}:{timestamp}"
        signature = self._compute_signature(message)

        result = self._make_request("check_credits", {
            "license_key": license_key,
            "frames_needed": amount_needed,
            "timestamp": timestamp,
            "signature": signature,
        })

        if "error" in result:
            return False, CreditInfo(error_message=result.get("error"))

        has_enough = result.get("has_enough", False)
        credits = CreditInfo(
            credits_remaining=result.get("credits_remaining", 0),
            is_subscription=result.get("is_subscription", False),
        )

        return has_enough, credits

    def deduct_credits(self, amount: int, license_key: str = None) -> Tuple[bool, CreditInfo]:
        """Deduct credits after processing."""
        license_key = license_key or self._license_key
        if not license_key:
            return False, CreditInfo(error_message="No license key")

        if amount <= 0:
            return True, self.get_credits(license_key)

        timestamp = int(time.time() * 1000)
        message = f"deduct_credits:{license_key}:{amount}:{timestamp}"
        signature = self._compute_signature(message)

        result = self._make_request("deduct_credits", {
            "license_key": license_key,
            "frames_processed": amount,
            "timestamp": timestamp,
            "signature": signature,
        })

        if "error" in result:
            return False, CreditInfo(error_message=result.get("error"))

        success = result.get("success", False)
        credits = CreditInfo(
            credits_remaining=result.get("credits_remaining", 0),
            is_subscription=result.get("is_subscription", False),
        )

        return success, credits


# Singleton instance
_client: Optional[SecureCreditClient] = None

def get_credit_client() -> SecureCreditClient:
    global _client
    if _client is None:
        _client = SecureCreditClient()
    return _client
```

### 4. `license_manager.py` - High-Level Manager

This wraps everything into an easy-to-use interface:

```python
"""
License manager - handles validation and credit tracking.
"""
import json
import os
import logging
from pathlib import Path
from typing import Optional, Tuple

from .lemon_squeezy import LemonSqueezyClient, LicenseInfo
from .usage_client import SecureCreditClient, CreditInfo, get_credit_client

logger = logging.getLogger(__name__)

DEFAULT_CACHE_DIR = Path.home() / ".myapp"
DEFAULT_CACHE_FILE = "license.json"

# Credit configuration - match your Lemon Squeezy variants
CREDIT_PACKS = {
    "Default": 100,
    "100 Credits": 100,
    "500 Credits": 500,
    "1000 Credits": 1000,
    "Weekly Subscription": 999999999,
}
DEFAULT_CREDITS = 100
SUBSCRIPTION_VARIANTS = ["Weekly Subscription", "Daily Subscription"]

# Set to False to use Cloudflare Worker for all validation
USE_DIRECT_VALIDATION = True


class LicenseManager:
    """License management with server-side credit tracking."""

    def __init__(self, cache_dir: Optional[Path] = None):
        self.cache_dir = cache_dir or DEFAULT_CACHE_DIR
        self.cache_path = self.cache_dir / DEFAULT_CACHE_FILE
        self.client = LemonSqueezyClient("")
        self.license_info: Optional[LicenseInfo] = None
        self.credit_client = get_credit_client()
        self._cached_credits: Optional[CreditInfo] = None
        self.cache_dir.mkdir(parents=True, exist_ok=True)

    def activate_license(self, license_key: str, instance_name: str = "MyApp") -> LicenseInfo:
        """Validate and activate a license key."""
        if USE_DIRECT_VALIDATION:
            return self._activate_direct(license_key, instance_name)
        else:
            return self._activate_via_cloudflare(license_key, instance_name)

    def _activate_direct(self, license_key: str, instance_name: str) -> LicenseInfo:
        """Activate directly with LemonSqueezy (no Cloudflare)."""
        self.license_info = self.client.activate_license(license_key, instance_name)

        if self.license_info.valid:
            if self._is_subscription_variant(self.license_info.variant_name):
                self._cached_credits = CreditInfo(
                    credits_total=999999999, credits_used=0, is_subscription=True
                )
            else:
                self._cached_credits = CreditInfo(
                    credits_total=DEFAULT_CREDITS, credits_used=0, is_subscription=False
                )
            self._save_cache()

        return self.license_info

    def _activate_via_cloudflare(self, license_key: str, instance_name: str) -> LicenseInfo:
        """Activate through Cloudflare Worker."""
        result = self.credit_client.activate(license_key, instance_name)

        self.license_info = LicenseInfo(
            valid=result.valid,
            license_key=license_key,
            instance_id=result.instance_id,
            customer_email=result.customer_email,
            customer_name=result.customer_name,
            product_name=result.product_name,
            variant_name=result.variant_name,
            activation_limit=result.activation_limit,
            activation_usage=result.activation_usage,
            error_message=result.error_message,
        )

        if result.valid:
            self._cached_credits = result.credits
            self._save_cache()

        return self.license_info

    def _is_subscription_variant(self, variant_name: str) -> bool:
        if variant_name in SUBSCRIPTION_VARIANTS:
            return True
        for sub_variant in SUBSCRIPTION_VARIANTS:
            if sub_variant.lower() in variant_name.lower():
                return True
        return False

    def load_cached_license(self) -> Optional[LicenseInfo]:
        """Load and validate cached license."""
        cached_data = self._load_cache_data()
        if not cached_data:
            return None

        license_key = cached_data.get("license_key")
        if not license_key:
            return None

        self.license_info = self.client.validate_license(license_key)

        if self.license_info.valid:
            self.license_info.instance_id = cached_data.get("instance_id")
            self.license_info.customer_email = cached_data.get("customer_email", "")
            self.license_info.customer_name = cached_data.get("customer_name", "")
            self.license_info.variant_name = cached_data.get("variant_name", "")
            self.license_info.expires_at = cached_data.get("expires_at")

            if self._is_subscription_variant(self.license_info.variant_name):
                self._cached_credits = CreditInfo(
                    credits_total=999999999, credits_used=0, is_subscription=True
                )
            elif not USE_DIRECT_VALIDATION:
                self._cached_credits = self.credit_client.get_credits(license_key)

            return self.license_info

        return None

    def _load_cache_data(self) -> dict:
        if not self.cache_path.exists():
            return {}
        try:
            with open(self.cache_path, 'r') as f:
                return json.load(f)
        except Exception:
            return {}

    def _save_cache(self):
        if not self.license_info:
            return
        cache_data = {
            "license_key": self.license_info.license_key,
            "instance_id": self.license_info.instance_id,
            "customer_email": self.license_info.customer_email,
            "customer_name": self.license_info.customer_name,
            "variant_name": self.license_info.variant_name,
            "expires_at": self.license_info.expires_at,
        }
        try:
            with open(self.cache_path, 'w') as f:
                json.dump(cache_data, f, indent=2)
        except Exception as e:
            logger.exception("Failed to save cache")

    def clear_cache(self):
        """Clear cached license (logout)."""
        if self.license_info and self.license_info.instance_id:
            self.client.deactivate_license(
                self.license_info.license_key,
                self.license_info.instance_id
            )
        if self.cache_path.exists():
            self.cache_path.unlink()
        self.license_info = None
        self._cached_credits = None

    def check_credits(self, amount_needed: int) -> Tuple[bool, int, str]:
        """Check if user has enough credits."""
        if not self.license_info or not self.license_info.valid:
            return (False, 0, "No valid license")

        if self._cached_credits and self._cached_credits.is_subscription:
            return (True, -1, "Subscription: Unlimited")

        if USE_DIRECT_VALIDATION:
            available = self._cached_credits.credits_remaining if self._cached_credits else 0
            return (True, available, f"Credits: {available:,}")

        has_enough, credit_info = self.credit_client.check_credits(
            amount_needed, self.license_info.license_key
        )
        self._cached_credits = credit_info

        if credit_info.is_subscription:
            return (True, -1, "Subscription: Unlimited")

        return (has_enough, credit_info.credits_remaining,
                f"Credits: {credit_info.credits_remaining:,}")

    def deduct_credits(self, amount: int) -> bool:
        """Deduct credits after processing."""
        if amount <= 0 or not self.license_info:
            return True

        if self.is_subscription:
            return True

        if USE_DIRECT_VALIDATION:
            return True

        success, credit_info = self.credit_client.deduct_credits(
            amount, self.license_info.license_key
        )
        if success:
            self._cached_credits = credit_info
        return success

    @property
    def is_licensed(self) -> bool:
        return self.license_info is not None and self.license_info.valid

    @property
    def is_subscription(self) -> bool:
        if self._cached_credits and self._cached_credits.is_subscription:
            return True
        if self.license_info:
            return self._is_subscription_variant(self.license_info.variant_name)
        return False

    @property
    def credits_remaining(self) -> int:
        if self._cached_credits:
            return self._cached_credits.credits_remaining
        return 0

    def get_status_display(self) -> str:
        if not self.license_info:
            return "Not logged in"
        if not self.license_info.valid:
            return f"Invalid: {self.license_info.error_message}"
        if self.license_info.customer_email:
            return f"Logged in as {self.license_info.customer_email}"
        return "Licensed"

    def get_credits_display(self) -> str:
        if not self.license_info or not self.license_info.valid:
            return "No license"
        if self.is_subscription:
            days = self.license_info.days_remaining
            if days is not None:
                return f"{days} Days Remaining"
            return "Subscription Active"
        return f"Credits: {self.credits_remaining:,}"
```

---

## Cloudflare Worker (Server-Side)

Create `cloudflare_worker.js`. See the Cloudflare Worker documentation for setup instructions.

**Setup**:
1. Cloudflare Dashboard > Workers & Pages > Create Worker
2. Add KV namespace: "MYAPP_CREDITS"
3. Add environment variables:
   - `LEMON_API_KEY`: Your Lemon Squeezy API key (encrypt)
   - `MYAPP_SECRET`: Random signing secret (encrypt)
   - `STORE_ID`: Your Lemon Squeezy store ID
   - `PRODUCT_ID`: Your product ID
   - `WEBHOOK_SECRET`: Lemon Squeezy webhook secret (encrypt)
4. Bind KV namespace: CREDITS -> MYAPP_CREDITS

The worker handles:
- License activation with Lemon Squeezy API
- Credit tracking in KV storage
- Webhook processing for new purchases
- HMAC-SHA256 signature verification
- Store/product ID validation

---

## PyInstaller Configuration

Add the licensing module to your `.spec` file:

```python
hiddenimports=[
    'utils.licensing',
    'utils.licensing.license_manager',
    'utils.licensing.lemon_squeezy',
    'utils.licensing.usage_client',
],
```

---

## Security Considerations

### 1. API Key Protection
- **Never** embed your Lemon Squeezy API key in distributed apps
- Store it only in Cloudflare Worker environment variables (encrypted)

### 2. Request Signing
- HMAC-SHA256 signatures prevent unauthorized API calls
- Signing secret in app is only for signing, not API access
- Even if extracted, attackers need valid license keys

### 3. Timestamp Validation
- 5-minute window prevents replay attacks
- Old requests are rejected

### 4. Server-Side Credits
- Credits stored in Cloudflare KV, not locally
- Users cannot tamper with their credit balance

### 5. Store/Product Verification
- License validated against your specific store and product IDs
- Prevents using licenses from other Lemon Squeezy products

---

## Quick Start Checklist

1. [ ] Create Lemon Squeezy account and product
2. [ ] Enable license keys for product
3. [ ] Create credit pack and/or subscription variants
4. [ ] Copy the Python files to `utils/licensing/`
5. [ ] (Optional) Set up Cloudflare Worker for credit tracking
6. [ ] Configure webhooks in Lemon Squeezy
7. [ ] Update constants (STORE_ID, WORKER_URL, etc.)
8. [ ] Integrate into your UI
9. [ ] Add to PyInstaller hidden imports
10. [ ] Test activation flow

---

## Troubleshooting

### "Activation limit reached"
- User has activated on too many devices
- Solution: Call `deactivate_license()` on old devices or increase limit in Lemon Squeezy

### "Invalid signature"
- Signing secret mismatch between app and Cloudflare Worker
- Check `MYAPP_SECRET` in Worker matches `SIGNING_SECRET` in Python

### "No credits found"
- License activated before Cloudflare KV was set up
- Solution: Use webhook to add credits, or manually add via `add_credits` action

### License not persisting
- Check cache directory permissions (`~/.myapp/`)
- Ensure `_save_cache()` is called after activation
