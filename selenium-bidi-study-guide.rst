=====================================
WebDriver BiDi with Selenium (Python)
=====================================

.. contents:: Table of Contents
   :depth: 2
   :local:

.. note::
   The BiDi API in Selenium is still evolving. Method names and modules can
   change between releases. Always verify examples against the docs for your
   installed version (``pip show selenium``).


1. What is WebDriver BiDi?
==========================

Classic WebDriver is **request/response**: your script sends a command over
HTTP, the browser replies, and nothing happens in between. The browser cannot
tell your script that something changed.

**WebDriver BiDi** (Bidirectional) keeps a **WebSocket** open between the test
and the browser. Both sides can send messages at any time, so the browser can
*push events* to your script (console logs, network traffic, navigation, ...).

It is a W3C standard being developed with the major browser vendors, and it is
meant to replace the Chromium-only Chrome DevTools Protocol (CDP) for these use
cases.


2. Classic vs CDP vs BiDi
=========================

.. list-table::
   :header-rows: 1
   :widths: 20 25 25 30

   * - Feature
     - WebDriver Classic
     - CDP
     - WebDriver BiDi
   * - Transport
     - HTTP request/response
     - WebSocket
     - WebSocket
   * - Direction
     - One way at a time
     - Two way
     - Two way
   * - Browser support
     - All major browsers
     - Chromium only
     - Cross-browser (growing)
   * - Standardised
     - Yes (W3C)
     - No (Chrome protocol)
     - Yes (W3C, in progress)
   * - Stability
     - Stable
     - Can change between versions
     - Standardised, still evolving


3. Architecture
===============

.. code-block:: text

   Test script (Python)
        |
        |  Selenium client (driver.script, driver.network, ...)
        v
   WebSocket  <----------------------------->  Browser driver
   (webSocketUrl)      commands + events       (chromedriver / geckodriver)
                                                     |
                                                     v
                                                  Browser

Key ideas:

* **Commands** go from the script to the browser and get a response with a
  matching ``id``.
* **Events** are pushed from the browser. You must **subscribe** first.
* Functionality is grouped into **modules** (also called domains).

Raw protocol messages look like this:

.. code-block:: json

   {"id": 1, "method": "session.subscribe",
    "params": {"events": ["log.entryAdded"]}}

   {"type": "event", "method": "log.entryAdded",
    "params": {"level": "info", "text": "hello"}}


4. Enabling BiDi in Python
==========================

BiDi is enabled through the ``webSocketUrl`` capability. In Python, use the
``enable_bidi`` option:

.. code-block:: python

   from selenium import webdriver

   options = webdriver.ChromeOptions()
   options.enable_bidi = True          # sets the webSocketUrl capability

   driver = webdriver.Chrome(options=options)

Firefox works the same way with ``webdriver.FirefoxOptions()``.

.. tip::
   If BiDi APIs fail with a connection error, check first that the capability
   is enabled and that your browser, driver, and Selenium versions are up to
   date and compatible.


5. BiDi modules
===============

The Python bindings expose these modules under
``selenium.webdriver.common.bidi``:

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Module
     - Purpose
   * - ``session``
     - Session status, subscribe and unsubscribe to events
   * - ``browsing_context``
     - Tabs and windows: create, navigate, reload, close, get the context tree
   * - ``script``
     - Run JavaScript, listen to console messages and JavaScript errors
   * - ``log``
     - Log entries (console, JavaScript, other)
   * - ``network``
     - Observe, intercept, and modify requests and responses
   * - ``storage``
     - Read and set cookies
   * - ``browser``
     - Browser-level operations such as user contexts
   * - ``webextension``
     - Install and uninstall browser extensions
   * - ``cdp``, ``console``
     - Older CDP-based helpers, kept for backward compatibility


6. Examples
===========

6.1 Capture console messages
----------------------------

.. code-block:: python

   from selenium import webdriver
   from selenium.webdriver.common.by import By
   from selenium.webdriver.support.ui import WebDriverWait

   options = webdriver.ChromeOptions()
   options.enable_bidi = True
   driver = webdriver.Chrome(options=options)

   log_entries = []
   driver.script.add_console_message_handler(log_entries.append)

   driver.get("https://www.selenium.dev/selenium/web/bidi/logEntryAdded.html")
   driver.find_element(By.ID, "consoleLog").click()

   WebDriverWait(driver, 5).until(lambda _: len(log_entries) > 0)
   print(log_entries[0].text)

   driver.quit()

6.2 Capture JavaScript errors
-----------------------------

.. code-block:: python

   js_errors = []
   driver.script.add_javascript_error_handler(js_errors.append)

   driver.get("https://www.selenium.dev/selenium/web/bidi/logEntryAdded.html")
   driver.find_element(By.ID, "jsException").click()

   WebDriverWait(driver, 5).until(lambda _: len(js_errors) > 0)
   assert "Error" in js_errors[0].text

6.3 Network monitoring and interception
---------------------------------------

The ``network`` module lets you watch requests and responses, block or mock
calls, and handle authentication prompts. Its API has changed across recent
releases, so read the current Selenium Python API docs before writing code.

.. warning::
   Old tutorials use ``driver.bidi_connection()`` together with ``trio`` and
   CDP wrappers. Prefer the newer ``driver.script`` / ``driver.network`` style
   APIs for new work.


7. Common use cases
===================

* Fail a test when the page logs a JavaScript error
* Assert on console output during a flow
* Wait for network activity to finish instead of using ``time.sleep``
* Mock or block API calls (offline mode, error responses, slow responses)
* Capture request and response data for debugging or performance checks
* Handle basic authentication without popups
* Manage cookies and multiple tabs more precisely


8. Best practices and pitfalls
==============================

* **Subscribe before you trigger the action**, or you will miss the event.
* Events arrive **asynchronously**. Use explicit waits, not ``sleep``.
* Register handlers once, and remove them when done to avoid duplicate
  callbacks.
* Keep event handlers fast. Collect data in a list and assert in the test.
* Pin and test browser, driver, and Selenium versions together.
* Do not assume every browser supports every BiDi module. Check support and
  add fallbacks or skip markers.
* For new code, prefer BiDi over ``execute_cdp_cmd`` so tests are not tied to
  Chromium.


9. Interview questions
======================

**What is WebDriver BiDi and why was it introduced?**
   A W3C standard for two-way communication over a WebSocket. Classic
   WebDriver cannot receive browser events, and CDP works only on Chromium.
   BiDi gives a cross-browser way to stream events and control the browser.

**How is BiDi different from CDP?**
   CDP is a Chrome-specific protocol that can change between versions. BiDi is
   a standardised, cross-browser protocol supported by multiple vendors.

**How do you enable BiDi in Selenium Python?**
   Set ``options.enable_bidi = True`` (the ``webSocketUrl`` capability) before
   creating the driver.

**Give a real use case.**
   Listen for console errors and JavaScript exceptions during a UI test and
   fail the test if any appear, instead of only checking the visible page.

**Does BiDi replace WebDriver Classic?**
   Not immediately. Selenium is migrating its internals toward BiDi while
   keeping backward compatibility, so both coexist.

**What could make BiDi tests flaky?**
   Subscribing too late, using fixed sleeps, duplicated handlers, and version
   mismatches between browser, driver, and Selenium.


10. Practice exercises
======================

1. Enable BiDi and print every console message from a page of your choice.
2. Write a pytest fixture that collects JavaScript errors and fails the test if
   the list is not empty at teardown.
3. Open a second tab using the ``browsing_context`` module and switch between
   tabs.
4. Add a network handler that logs the URL of every request during a login
   flow.
5. Run the same BiDi test on Chrome and Firefox and note any differences.


11. References
==============

* Selenium docs, BiDirectional functionality:
  https://www.selenium.dev/documentation/webdriver/bidi/
* Selenium Python API docs (``selenium.webdriver.common.bidi``):
  https://www.selenium.dev/selenium/docs/api/py/
* W3C WebDriver BiDi specification:
  https://w3c.github.io/webdriver-bidi/
