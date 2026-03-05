# Pixashot + ColourThief Integration — IDE Prompt

## Context
This is a forked version of Pixashot (open source screenshot service) hosted at:
`https://github.com/riddhipanchal698/pixashoot`

We need to modify it to also extract the top 5 dominant colors from every screenshot it takes, and return those colors in the API response alongside the screenshot.

---

## What Pixashot Does
Pixashot is a Python + Playwright based screenshot API. It:
- Receives a POST request to `/capture` with a URL
- Opens a headless browser, navigates to the URL
- Takes a screenshot and saves it to disk
- Returns the screenshot as a response

It is built with:
- **Python** (async)
- **Playwright** (browser automation)
- **Quart** (async web framework)

---

## What We Want to Add
After the screenshot is taken and saved to disk, we want to:
1. Run **ColourThief** (Python library) on the saved screenshot
2. Extract the **top 5 dominant colors** as hex codes e.g. `["#ff5733", "#2c3e50", "#ffffff", "#f39c12", "#27ae60"]`
3. Include the `colors` array in the **JSON API response** returned to the caller

---

## Files to Modify

### 1. `requirements.txt`
Add this line:
```
colorthief
```

---

### 2. `src/capture_service.py`
This is where the screenshot is taken. The method `capture_screenshot()` currently returns just `intermediate_format`.

**We need to:**
- Import `ColorThief` from `colorthief` and `BytesIO` from `io`
- Add a `extract_colors(image_path, color_count=5)` helper function after imports
- After the screenshot is saved to `output_path`, call `extract_colors(output_path)`
- Change the return value from just `intermediate_format` to `(intermediate_format, colors)`

**Here is the current `capture_service.py` for reference:**
```python
import logging
from playwright.async_api import Page
from exceptions import ScreenshotServiceException
from controllers.main_controller import MainBrowserController
from controllers.screenshot_controller import ScreenshotController
from context_manager import ContextManager

logger = logging.getLogger(__name__)

class CaptureService:
    def __init__(self):
        self.main_controller = None
        self.screenshot_controller = None
        self.context_manager = None
        self.context = None
        self.playwright = None

    async def initialize(self, playwright):
        self.playwright = playwright
        self.main_controller = MainBrowserController()
        self.screenshot_controller = ScreenshotController()
        self.context_manager = ContextManager()
        self.context = await self.context_manager.initialize(playwright)

    async def _configure_page(self, page, options):
        if getattr(options, 'use_random_user_agent', True):
            headers = self.context_manager._generate_headers(options)
            await page.set_extra_http_headers(headers)

    async def _resilient_navigation(self, page, url, timeout):
        try:
            await page.goto(str(url), wait_until='domcontentloaded', timeout=timeout)
        except Exception as nav_error:
            logger.warning(f"Navigation timeout or error: {str(nav_error)}. Continuing...")
            try:
                await page.wait_for_timeout(1000)
            except Exception as wait_error:
                logger.warning(f"Additional wait failed: {str(wait_error)}")

    async def capture_screenshot(self, output_path, options):
        page = None
        try:
            page = await self.context.new_page()
            try:
                await self._configure_page(page, options)
                await self.main_controller.prepare_page(page, options)
                if options.url:
                    await self._resilient_navigation(page, str(options.url), options.wait_for_timeout)
                else:
                    await page.set_content(options.html_content)
                if options.interactions:
                    await self.main_controller.perform_interactions(page, options.interactions)
                if options.full_page:
                    await self.main_controller.prepare_for_full_page_screenshot(page, options.window_width)
                else:
                    await self.main_controller.prepare_for_viewport_screenshot(page, options.window_width, options.window_height)
                intermediate_format = 'png' if options.format == 'webp' else options.format
                await self.screenshot_controller.take_screenshot(page, {
                    'path': output_path,
                    'full_page': options.full_page,
                    'format': intermediate_format,
                    'quality': options.image_quality if intermediate_format != 'png' else None,
                    'omit_background': options.omit_background
                })
                return intermediate_format  # <-- THIS NEEDS TO CHANGE to return (intermediate_format, colors)
            finally:
                await page.close()
        except Exception as e:
            logger.error(f"Screenshot capture error: {str(e)}")
            raise ScreenshotServiceException(str(e))

    async def close(self):
        if self.context_manager:
            await self.context_manager.close()
```

---

### 3. `src/routes.py`
This is where the API response is built and returned to the caller.

**We need to:**
- Update the call to `capture_service.capture_screenshot()` to unpack the new tuple return value `(format, colors)`
- Add `colors` to the JSON response

**You need to find where `capture_screenshot()` is called in `routes.py` and:**
1. Change `format = await capture_service.capture_screenshot(...)` to `format, colors = await capture_service.capture_screenshot(...)`
2. Include `colors` in the response JSON

---

## Expected API Response After Modification

When n8n calls `POST /capture` with a URL, the response should look like:

```json
{
  "status": "success",
  "screenshot": "<screenshot data or path>",
  "colors": ["#ff5733", "#2c3e50", "#ffffff", "#f39c12", "#27ae60"]
}
```

---

## Important Notes
- `extract_colors()` should handle errors gracefully — if color extraction fails for any reason, return an empty list `[]` and log a warning. Never let color extraction failure break the screenshot capture.
- ColourThief works on the saved file path, not in-memory bytes
- Keep the existing screenshot functionality 100% intact — we are only ADDING color extraction, not changing anything else
- The `colorthief` library returns RGB tuples like `(255, 87, 51)` — convert each to hex like `#ff5733`

---

## Summary of Changes
| File | Change |
|---|---|
| `requirements.txt` | Add `colorthief` |
| `src/capture_service.py` | Add `extract_colors()` function, modify `capture_screenshot()` to return `(format, colors)` |
| `src/routes.py` | Unpack new return value, add `colors` to response JSON |

After making changes, push to GitHub:
```bash
git add .
git commit -m "Add ColourThief color extraction to capture response"
git push origin develop
```