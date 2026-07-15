# Hướng dẫn đầy đủ: Kiểm thử tự động với Playwright + JUnit 5 (Java)

> Áp dụng cho cùng trang web của Lab 3: https://the-internet.herokuapp.com/login
> Stack: Java 17 + Maven + Playwright for Java + JUnit 5
> Cấu trúc bám theo Lab 3 (Selenium) để dễ so sánh 2 công cụ.

---

## 1. Playwright là gì? Khác gì Selenium?

Playwright là thư viện tự động hóa trình duyệt của Microsoft, hỗ trợ Chromium, Firefox, WebKit và các ngôn ngữ Java, TypeScript/JS, Python, C#.

| Tiêu chí | Selenium WebDriver | Playwright |
|----------|--------------------|------------|
| Driver | Cần driver riêng (chromedriver...) qua WebDriverManager | Tự tải browser khi cài, không cần driver |
| Wait | Phải tự viết Explicit Wait (`WebDriverWait`) | **Auto-wait** tích hợp sẵn cho mọi action |
| Assertion | JUnit `assertTrue(...)` + wait thủ công | `PlaywrightAssertions.assertThat(...)` tự retry |
| Tốc độ | Chậm hơn (giao tiếp qua WebDriver protocol) | Nhanh hơn (giao tiếp qua CDP/pipe) |
| Browser context | Mỗi driver 1 profile | `BrowserContext` nhẹ, cô lập, tạo nhanh |
| Debug | Screenshot thủ công | Codegen, Trace Viewer, screenshot/video tích hợp |
| Headless | `--headless` qua ChromeOptions | Mặc định headless, bật UI bằng `setHeadless(false)` |

Điểm mấu chốt khi chuyển từ Selenium sang: **không cần `WebDriverWait` / `ExpectedConditions` nữa** — `locator.click()`, `locator.fill()`, `assertThat()` tự chờ element sẵn sàng.

---

## 2. Cấu trúc dự án theo Page Object Model (đích cuối)

```
PlaywrightLogin
├── pom.xml
└── src
    └── test
        ├── java
        │   ├── pages
        │   │   ├── BasePage.java        # lớp cha: navigate, fill, click, getText
        │   │   └── LoginPage.java       # locator + action của trang Login
        │   ├── tests
        │   │   ├── BaseTest.java        # quản lý vòng đời Playwright/Browser/Context/Page
        │   │   └── LoginTest.java       # chỉ chứa logic kiểm thử
        │   └── utils
        │       └── BrowserFactory.java  # khởi tạo & cấu hình Browser dùng chung
        └── resources
            └── login-data.csv           # dữ liệu test bên ngoài
```

---

## TODO 1: Tạo project Maven + cấu hình pom.xml

**Các bước:**

1. IntelliJ → `New Project → Maven` → đặt tên `PlaywrightLogin`.
2. Thêm dependency vào `pom.xml` rồi `Maven → Reload Project`.
3. Lần chạy đầu, Playwright tự tải browser (Chromium, Firefox, WebKit). Có thể tải trước bằng:
   ```
   mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="install"
   ```

**Code đầy đủ – `pom.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>swt301</groupId>
    <artifactId>PlaywrightLogin</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- Playwright for Java (kiểm tra version mới nhất tại
             https://mvnrepository.com/artifact/com.microsoft.playwright/playwright) -->
        <dependency>
            <groupId>com.microsoft.playwright</groupId>
            <artifactId>playwright</artifactId>
            <version>1.45.0</version>
        </dependency>

        <!-- JUnit 5 -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-engine</artifactId>
            <version>5.10.2</version>
            <scope>test</scope>
        </dependency>

        <!-- JUnit 5 Parameterized Test -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-params</artifactId>
            <version>5.10.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

> Không cần WebDriverManager — Playwright tự quản lý browser.

**Checklist:**

- [ ] Project build thành công, browser được tải về (`~/.cache/ms-playwright`)
- [ ] Không còn dependency selenium/webdrivermanager

**Commit:**
```
chore: init maven project with playwright and junit5
```

---

## TODO 2: Viết `LoginTest.java` cơ bản (2 test case)

**Các bước:**

1. Tạo `LoginTest.java` trong `src/test/java`.
2. `@BeforeAll`: tạo `Playwright` → `Browser` (Chromium, có UI). `@BeforeEach`: tạo `BrowserContext` + `Page` mới cho mỗi test (cô lập, tương đương incognito). `@AfterEach`/`@AfterAll`: đóng lại.
3. Viết 2 test login thành công / thất bại — **không cần wait thủ công**.

**Code đầy đủ – `src/test/java/LoginTest.java`:**

```java
import com.microsoft.playwright.*;
import org.junit.jupiter.api.*;

import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@DisplayName("Login Tests for the-internet.herokuapp.com (Playwright)")
public class LoginTest {
    static Playwright playwright;
    static Browser browser;
    BrowserContext context;
    Page page;

    @BeforeAll
    static void launchBrowser() {
        playwright = Playwright.create();
        browser = playwright.chromium().launch(
                new BrowserType.LaunchOptions()
                        .setHeadless(false)   // false: hiện UI; true (mặc định): chạy ngầm cho CI/CD
                        .setSlowMo(200)       // làm chậm 200ms mỗi thao tác để quan sát (bỏ khi chạy thật)
        );
    }

    @BeforeEach
    void createContext() {
        // Mỗi test 1 context mới => cô lập cookie/session, tương đương chế độ ẩn danh
        context = browser.newContext();
        page = context.newPage();
    }

    @Test
    @Order(1)
    @DisplayName("Should login successfully with valid credentials")
    void testLoginSuccess() {
        page.navigate("https://the-internet.herokuapp.com/login");

        page.locator("#username").fill("tomsmith");
        page.locator("#password").fill("SuperSecretPassword!");
        page.locator("button[type='submit']").click();

        // assertThat tự động chờ (auto-wait + retry) đến khi element hiển thị đúng nội dung
        assertThat(page.locator(".flash.success"))
                .containsText("You logged into a secure area!");
    }

    @Test
    @Order(2)
    @DisplayName("Should display error when logging in with invalid credentials")
    void testLoginFail() {
        page.navigate("https://the-internet.herokuapp.com/login");

        page.locator("#username").fill("invalid");
        page.locator("#password").fill("wrongpassword");
        page.locator("button[type='submit']").click();

        assertThat(page.locator(".flash.error"))
                .containsText("Your username is invalid!");
    }

    @AfterEach
    void closeContext() {
        context.close();
    }

    @AfterAll
    static void closeBrowser() {
        browser.close();
        playwright.close();
    }
}
```

So sánh với Selenium:

| Selenium | Playwright |
|----------|-----------|
| `driver.get(url)` | `page.navigate(url)` |
| `driver.findElement(By.id("username")).sendKeys(...)` | `page.locator("#username").fill(...)` |
| `wait.until(ExpectedConditions.visibilityOfElementLocated(...))` rồi `assertTrue(...)` | `assertThat(locator).containsText(...)` (tự chờ) |
| `driver.quit()` | `browser.close(); playwright.close()` |

**Checklist:**

- [ ] Không có `Thread.sleep()` và không cần `WebDriverWait`
- [ ] Mỗi test dùng `BrowserContext` mới
- [ ] 2 test đều pass

**Commit:**
```
test: add basic login success and failure tests using playwright
```

---

## TODO 3: `@ParameterizedTest` + `@CsvSource`

JUnit 5 dùng y hệt Lab 3 — chỉ đổi phần thao tác trang sang Playwright.

**Code đầy đủ – thêm vào `LoginTest.java`:**

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

@Order(3)
@ParameterizedTest(name = "Test Login - Username: {0}, Password: {1}")
@CsvSource({
        "tomsmith, SuperSecretPassword!, success",         // valid
        "wronguser, SuperSecretPassword!, error",          // invalid username
        "tomsmith, wrongpassword, error",                  // invalid password
        "'', '', error"                                    // empty credentials
})
@DisplayName("Multiple login attempts using @CsvSource")
void testLoginWithMultipleParameters(String username, String password, String expectedResult) {
    page.navigate("https://the-internet.herokuapp.com/login");

    page.locator("#username").fill(username);
    page.locator("#password").fill(password);
    page.locator("button[type='submit']").click();

    if (expectedResult.equals("success")) {
        assertThat(page.locator(".flash.success"))
                .containsText("You logged into a secure area!");
    } else {
        assertThat(page.locator(".flash.error")).containsText("invalid");
    }
}
```

**Checklist:**

- [ ] Chạy đủ 4 bộ dữ liệu, tên test hiển thị theo `name = "..."`
- [ ] Cover cả positive lẫn negative case

**Commit:**
```
test: add parameterized login test with @CsvSource using playwright
```

---

## TODO 4: Đọc dữ liệu test từ file CSV bên ngoài

**Các bước:**

1. Tạo thư mục `src/test/resources`, thêm file `login-data.csv` (giống Lab 3):

```csv
username,password,expectedResult
tomsmith,SuperSecretPassword!,success
wronguser,SuperSecretPassword!,error
tomsmith,wrongpassword,error
, ,error
```

2. Thêm test dùng `@CsvFileSource`:

**Code đầy đủ – thêm vào `LoginTest.java`:**

```java
import org.junit.jupiter.params.provider.CsvFileSource;

@Order(4)
@ParameterizedTest(name = "Test login with: {0} / {1}")
@CsvFileSource(resources = "/login-data.csv", numLinesToSkip = 1)
@DisplayName("Login with data from external CSV file")
void testLoginWithCSV(String username, String password, String expectedResult) {
    page.navigate("https://the-internet.herokuapp.com/login");

    // Chuyển null thành chuỗi rỗng + trim() tránh lỗi khoảng trắng khi copy/paste CSV
    username = (username == null) ? "" : username.trim();
    password = (password == null) ? "" : password.trim();

    page.locator("#username").fill(username);
    page.locator("#password").fill(password);
    page.locator("button[type='submit']").click();

    if (expectedResult.equals("success")) {
        assertThat(page.locator(".flash.success"))
                .containsText("You logged into a secure area!");
    } else {
        assertThat(page.locator(".flash.error")).containsText("invalid");
    }
}
```

**Checklist:**

- [ ] `login-data.csv` nằm trong `src/test/resources`
- [ ] Có `numLinesToSkip = 1`, xử lý `null` + `trim()`
- [ ] Test chạy đủ số dòng CSV

**Commit:**
```
test: add data-driven login test reading from external csv file
```

---

## TODO 5: Tổ chức theo Page Object Model

**Các bước:**

1. Tạo 3 package: `pages`, `tests`, `utils`.
2. Tạo `BrowserFactory`, `LoginPage`, chuyển test sang dùng Page Object.

**Code đầy đủ – `src/test/java/utils/BrowserFactory.java`:**

```java
package utils;

import com.microsoft.playwright.*;

public class BrowserFactory {
    public static Playwright createPlaywright() {
        return Playwright.create();
    }

    public static Browser createBrowser(Playwright playwright) {
        return playwright.chromium().launch(
                new BrowserType.LaunchOptions()
                        .setHeadless(false)  // đổi true khi chạy CI/CD
        );
        // Đổi browser chỉ cần: playwright.firefox().launch(...) hoặc playwright.webkit().launch(...)
    }
}
```

**Code đầy đủ – `src/test/java/pages/LoginPage.java`:**

```java
package pages;

import com.microsoft.playwright.Locator;
import com.microsoft.playwright.Page;

public class LoginPage {
    private final Page page;

    // Locators
    private final Locator usernameField;
    private final Locator passwordField;
    private final Locator loginButton;
    private final Locator successMsg;
    private final Locator errorMsg;

    public LoginPage(Page page) {
        this.page = page;
        this.usernameField = page.locator("#username");
        this.passwordField = page.locator("#password");
        this.loginButton = page.locator("button[type='submit']");
        this.successMsg = page.locator(".flash.success");
        this.errorMsg = page.locator(".flash.error");
    }

    // Actions
    public void navigate() {
        page.navigate("https://the-internet.herokuapp.com/login");
    }

    public void login(String username, String password) {
        usernameField.fill(username);
        passwordField.fill(password);
        loginButton.click();
    }

    public Locator getSuccessMsg() {
        return successMsg;
    }

    public Locator getErrorMsg() {
        return errorMsg;
    }
}
```

> Khác Selenium: Page Object trả về `Locator` (đã gắn auto-wait) thay vì `By`, nên test không cần wait.

**Code đầy đủ – `src/test/java/tests/LoginTest.java`:**

```java
package tests;

import com.microsoft.playwright.*;
import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvFileSource;
import org.junit.jupiter.params.provider.CsvSource;
import pages.LoginPage;
import utils.BrowserFactory;

import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@DisplayName("Login Tests using Page Object Model (Playwright)")
public class LoginTest {
    static Playwright playwright;
    static Browser browser;
    BrowserContext context;
    Page page;
    LoginPage loginPage;

    @BeforeAll
    static void launchBrowser() {
        playwright = BrowserFactory.createPlaywright();
        browser = BrowserFactory.createBrowser(playwright);
    }

    @BeforeEach
    void createContextAndPage() {
        context = browser.newContext();
        page = context.newPage();
        loginPage = new LoginPage(page);
    }

    @Test
    @Order(1)
    @DisplayName("Should login successfully with valid credentials")
    void testLoginSuccess() {
        loginPage.navigate();
        loginPage.login("tomsmith", "SuperSecretPassword!");
        assertThat(loginPage.getSuccessMsg()).containsText("You logged into a secure area!");
    }

    @Test
    @Order(2)
    @DisplayName("Should show error for invalid credentials")
    void testLoginFail() {
        loginPage.navigate();
        loginPage.login("wronguser", "wrongpassword");
        assertThat(loginPage.getErrorMsg()).containsText("invalid");
    }

    @ParameterizedTest(name = "CSV Inline: {0} / {1}")
    @Order(3)
    @CsvSource({
            "tomsmith, SuperSecretPassword!, success",
            "wronguser, SuperSecretPassword!, error",
            "tomsmith, wrongpassword, error",
            "'', '', error"
    })
    void testLoginCsvInline(String username, String password, String expected) {
        loginPage.navigate();
        username = (username == null) ? "" : username.trim();
        password = (password == null) ? "" : password.trim();

        loginPage.login(username, password);

        if (expected.equals("success")) {
            assertThat(loginPage.getSuccessMsg()).containsText("You logged into a secure area!");
        } else {
            assertThat(loginPage.getErrorMsg()).containsText("invalid");
        }
    }

    @ParameterizedTest(name = "CSV File: {0} / {1}")
    @Order(4)
    @CsvFileSource(resources = "/login-data.csv", numLinesToSkip = 1)
    void testLoginFromCSV(String username, String password, String expected) {
        loginPage.navigate();
        username = (username == null) ? "" : username.trim();
        password = (password == null) ? "" : password.trim();

        loginPage.login(username, password);

        if (expected.equals("success")) {
            assertThat(loginPage.getSuccessMsg()).containsText("You logged into a secure area!");
        } else {
            assertThat(loginPage.getErrorMsg()).containsText("invalid");
        }
    }

    @AfterEach
    void closeContext() {
        context.close();
    }

    @AfterAll
    static void closeBrowser() {
        browser.close();
        playwright.close();
    }
}
```

**Checklist:**

- [ ] Test class không gọi trực tiếp `page.locator(...)`
- [ ] Locator để `private final` trong Page class
- [ ] Cả 4 nhóm test (Order 1→4) đều pass

**Commit (theo từng file):**
```
feat: add BrowserFactory for shared playwright browser configuration
feat: add LoginPage page object with playwright locators and actions
refactor: migrate playwright login tests to page object model
```

---

## TODO 6: Xây dựng `BasePage` và `BaseTest`

**Code đầy đủ – `src/test/java/pages/BasePage.java`:**

```java
package pages;

import com.microsoft.playwright.Locator;
import com.microsoft.playwright.Page;

public class BasePage {
    protected Page page;

    public BasePage(Page page) {
        this.page = page;
    }

    // Navigate to a URL
    public void navigateTo(String url) {
        page.navigate(url);
    }

    // Fill safely (Playwright tự chờ element sẵn sàng)
    protected void type(Locator locator, String text) {
        locator.fill(text);
    }

    // Click safely
    protected void click(Locator locator) {
        locator.click();
    }

    // Get text safely
    protected String getText(Locator locator) {
        return locator.textContent();
    }

    // Check if element is visible
    protected boolean isElementVisible(Locator locator) {
        return locator.isVisible();
    }
}
```

**Code đầy đủ – `src/test/java/pages/LoginPage.java` (kế thừa BasePage):**

```java
package pages;

import com.microsoft.playwright.Locator;
import com.microsoft.playwright.Page;

public class LoginPage extends BasePage {

    private final Locator usernameField;
    private final Locator passwordField;
    private final Locator loginButton;
    private final Locator successMsg;
    private final Locator errorMsg;

    public LoginPage(Page page) {
        super(page);
        this.usernameField = page.locator("#username");
        this.passwordField = page.locator("#password");
        this.loginButton = page.locator("button[type='submit']");
        this.successMsg = page.locator(".flash.success");
        this.errorMsg = page.locator(".flash.error");
    }

    public void navigate() {
        navigateTo("https://the-internet.herokuapp.com/login");
    }

    public void login(String username, String password) {
        type(usernameField, username);
        type(passwordField, password);
        click(loginButton);
    }

    public Locator getSuccessMsg() {
        return successMsg;
    }

    public Locator getErrorMsg() {
        return errorMsg;
    }
}
```

**Code đầy đủ – `src/test/java/tests/BaseTest.java`:**

```java
package tests;

import com.microsoft.playwright.*;
import org.junit.jupiter.api.*;
import utils.BrowserFactory;

public abstract class BaseTest {
    protected static Playwright playwright;
    protected static Browser browser;
    protected BrowserContext context;
    protected Page page;

    @BeforeAll
    public static void setUpBase() {
        playwright = BrowserFactory.createPlaywright();
        browser = BrowserFactory.createBrowser(playwright);
    }

    @BeforeEach
    public void createContext() {
        context = browser.newContext();
        page = context.newPage();
    }

    @AfterEach
    public void closeContext() {
        if (context != null) {
            context.close();
        }
    }

    @AfterAll
    public static void tearDownBase() {
        if (browser != null) browser.close();
        if (playwright != null) playwright.close();
    }
}
```

**Code đầy đủ – `src/test/java/tests/LoginTest.java` (kế thừa BaseTest):**

```java
package tests;

import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvFileSource;
import org.junit.jupiter.params.provider.CsvSource;
import pages.LoginPage;

import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@DisplayName("Login Tests using POM + BaseTest (Playwright)")
public class LoginTest extends BaseTest {
    LoginPage loginPage;

    @BeforeEach
    void initPage() {
        loginPage = new LoginPage(page); // page do BaseTest tạo mỗi test
    }

    @Test
    @Order(1)
    @DisplayName("Should login successfully with valid credentials")
    void testLoginSuccess() {
        loginPage.navigate();
        loginPage.login("tomsmith", "SuperSecretPassword!");
        assertThat(loginPage.getSuccessMsg()).containsText("You logged into a secure area!");
    }

    @Test
    @Order(2)
    @DisplayName("Should show error for invalid credentials")
    void testLoginFail() {
        loginPage.navigate();
        loginPage.login("wronguser", "wrongpassword");
        assertThat(loginPage.getErrorMsg()).containsText("invalid");
    }

    @ParameterizedTest(name = "CSV Inline: {0} / {1}")
    @Order(3)
    @CsvSource({
            "tomsmith, SuperSecretPassword!, success",
            "wronguser, SuperSecretPassword!, error",
            "tomsmith, wrongpassword, error",
            "'', '', error"
    })
    void testLoginCsvInline(String username, String password, String expected) {
        loginPage.navigate();
        username = (username == null) ? "" : username.trim();
        password = (password == null) ? "" : password.trim();

        loginPage.login(username, password);

        if (expected.equals("success")) {
            assertThat(loginPage.getSuccessMsg()).containsText("You logged into a secure area!");
        } else {
            assertThat(loginPage.getErrorMsg()).containsText("invalid");
        }
    }

    @ParameterizedTest(name = "CSV File: {0} / {1}")
    @Order(4)
    @CsvFileSource(resources = "/login-data.csv", numLinesToSkip = 1)
    void testLoginFromCSV(String username, String password, String expected) {
        loginPage.navigate();
        username = (username == null) ? "" : username.trim();
        password = (password == null) ? "" : password.trim();

        loginPage.login(username, password);

        if (expected.equals("success")) {
            assertThat(loginPage.getSuccessMsg()).containsText("You logged into a secure area!");
        } else {
            assertThat(loginPage.getErrorMsg()).containsText("invalid");
        }
    }
}
```

> Lưu ý: `BaseTest` là `abstract`. Vì mỗi test có `page` mới (`@BeforeEach`), `LoginPage` cũng khởi tạo lại trong `@BeforeEach` — khác Lab 3 Selenium dùng `@BeforeAll` vì driver dùng chung.

**Checklist:**

- [ ] `BasePage` chứa: navigateTo, type, click, getText, isElementVisible
- [ ] `LoginPage` kế thừa `BasePage`
- [ ] `BaseTest` là `abstract`, quản lý vòng đời Playwright → Browser → Context → Page
- [ ] Tất cả test pass

**Commit (theo từng file):**
```
feat: add BasePage with reusable playwright interactions
refactor: make LoginPage extend BasePage
feat: add abstract BaseTest managing playwright lifecycle
refactor: make LoginTest extend BaseTest
```

---

## 3. Các tính năng hữu ích của Playwright (nên biết)

**Codegen – tự sinh code test bằng cách thao tác trên trình duyệt:**
```
mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="codegen https://the-internet.herokuapp.com/login"
```

**Screenshot:**
```java
page.screenshot(new Page.ScreenshotOptions()
        .setPath(java.nio.file.Paths.get("screenshots/login.png"))
        .setFullPage(true));
```

**Quay video mỗi test:**
```java
context = browser.newContext(new Browser.NewContextOptions()
        .setRecordVideoDir(java.nio.file.Paths.get("videos/")));
```

**Trace Viewer – ghi lại toàn bộ diễn biến test để debug khi fail:**
```java
context.tracing().start(new Tracing.StartOptions()
        .setScreenshots(true).setSnapshots(true));
// ... chạy test ...
context.tracing().stop(new Tracing.StopOptions()
        .setPath(java.nio.file.Paths.get("trace.zip")));
```
Xem trace: `mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="show-trace trace.zip"`

**Locator nên dùng (theo khuyến nghị Playwright):**

```java
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Login"));
page.getByLabel("Username");
page.getByPlaceholder("Enter password");
page.getByText("You logged into a secure area!");
page.locator("#username");            // CSS
page.locator("//input[@id='username']"); // XPath (hạn chế dùng)
```

**Chạy headless cho CI/CD:** bỏ `setHeadless(false)` (mặc định là headless) hoặc:

```java
playwright.chromium().launch(new BrowserType.LaunchOptions().setHeadless(true));
```

---

## 4. Lỗi thường gặp

| Lỗi | Nguyên nhân / Cách sửa |
|-----|------------------------|
| `Driver not found` / tải browser chậm lần đầu | Playwright đang tải browser; chạy lệnh `install` trước (mục TODO 1) |
| `TimeoutError 30000ms` khi click/fill | Locator sai hoặc element không xuất hiện — kiểm tra bằng codegen/trace; timeout mặc định 30s, chỉnh bằng `locator.click(new Locator.ClickOptions().setTimeout(10000))` |
| Test flaky khi dùng `page.textContent()` + `assertTrue` | Dùng `assertThat(locator).containsText(...)` để có auto-retry |
| Chạy CI Linux thiếu thư viện hệ thống | `mvn exec:java ... exec.args="install --with-deps"` |

---

## 5. Luồng commit đầy đủ (Conventional Commits)

```
chore: init maven project with playwright and junit5
test: add basic login success and failure tests using playwright
test: add parameterized login test with @CsvSource using playwright
test: add data-driven login test reading from external csv file
feat: add BrowserFactory for shared playwright browser configuration
feat: add LoginPage page object with playwright locators and actions
refactor: migrate playwright login tests to page object model
feat: add BasePage with reusable playwright interactions
refactor: make LoginPage extend BasePage
feat: add abstract BaseTest managing playwright lifecycle
refactor: make LoginTest extend BaseTest
```
