# Lab 3 – Kiểm thử tự động với Selenium WebDriver + JUnit 5

> Môn: SWT301 – Automation Testing
> Tài liệu gốc: *KiemThuTuDong voi Selenium Junit 5_traltb.docx* (traltb@fe.edu.vn)
> Website test: https://the-internet.herokuapp.com/login

---

## Mục tiêu

- Viết test tự động UI bằng Selenium WebDriver 4 + JUnit 5.
- Dùng WebDriverManager để tự cấu hình driver.
- Data-driven testing với `@ParameterizedTest`, `@CsvSource`, `@CsvFileSource`.
- Tổ chức dự án theo **Page Object Model (POM)** với `BasePage`, `BaseTest`, `DriverFactory`.
- Commit code lên GitHub theo chuẩn **Conventional Commits**.

## Yêu cầu môi trường

- JDK 17, Maven, IntelliJ IDEA
- Chrome browser
- (Tùy chọn) Plugin **CSV Editor** cho IntelliJ: `File → Settings → Plugins → gõ "CSV Editor" → Install → Restart IDE`

---

## Cấu trúc dự án theo Page Object Model (đích cuối – TODO 5, 6)

```
Exercise3_POM / Exercise4_BasePage
├── pom.xml
└── src
    └── test
        ├── java
        │   ├── pages
        │   │   ├── BasePage.java        # lớp cha: wait, click, type, getText...
        │   │   └── LoginPage.java       # locator + action của trang Login
        │   ├── tests
        │   │   ├── BaseTest.java        # abstract: @BeforeAll/@AfterAll khởi tạo/quit driver
        │   │   └── LoginTest.java       # chỉ chứa logic kiểm thử
        │   └── utils
        │       └── DriverFactory.java   # khởi tạo & cấu hình WebDriver dùng chung
        └── resources
            └── login-data.csv           # dữ liệu test bên ngoài
```

Nguyên tắc POM:

- Tách logic test và logic UI: mỗi trang là 1 class riêng.
- UI đổi → chỉ sửa 1 class Page. Dễ bảo trì, dễ tái sử dụng, teamwork/CI-CD tốt hơn.
- **Không** thao tác DOM trực tiếp trong test class (`driver.findElement...`) → luôn qua Page Object.
- **Không** dùng `Thread.sleep()` → dùng `WebDriverWait` + `ExpectedConditions`.

---

## TODO 1: Tạo project Maven `Exercise1_SeleniumBasic` + cấu hình pom.xml

**Các bước:**

1. IntelliJ → `New Project → Maven` → đặt tên `Exercise1_SeleniumBasic`.
2. Mở `pom.xml`, thêm properties + dependencies.
3. `Maven → Reload Project` để tải dependency.

**Code đầy đủ – `pom.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>traltb@fe.edu.vn</groupId>
    <artifactId>SeleniumDemo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- Selenium Java -->
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <version>4.21.0</version>
        </dependency>

        <!-- JUnit 5 -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-engine</artifactId>
            <version>5.10.2</version>
            <scope>test</scope>
        </dependency>

        <!-- WebDriverManager: tự tải driver đúng version, tự set path,
             hỗ trợ Chrome, Firefox, Edge, Opera... -->
     <dependency>
        <groupId>io.github.bonigarcia</groupId>
        <artifactId>webdrivermanager</artifactId>
        <version>6.1.0</version>
    </dependency>
    </dependencies>
</project>
```

**Checklist:**

- [ ] Project Maven build thành công (`mvn clean compile`)
- [ ] 3 dependency: selenium-java, junit-jupiter-engine, webdrivermanager
- [ ] JDK 17 được cấu hình đúng

**Commit:**
```
chore: init maven project with selenium, junit5 and webdrivermanager
```

---

## TODO 2: Viết `LoginTest.java` cơ bản (2 test case)

**Các bước:**

1. Tạo file `LoginTest.java` trong `src/test/java`.
2. `@BeforeAll`: setup WebDriverManager, ChromeOptions (`--incognito`, chặn JavaScript), khởi tạo driver + wait, maximize window.
3. Test 1 – login thành công. Test 2 – login thất bại.
4. `@AfterAll`: `driver.quit()`.

**Code đầy đủ – `src/test/java/LoginTest.java`:**

```java
import io.github.bonigarcia.wdm.WebDriverManager;
import org.junit.jupiter.api.*;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;
import java.util.HashMap;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.assertTrue;

@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@DisplayName("Login Tests for the-internet.herokuapp.com")
public class LoginTest {
    static WebDriver driver;
    static WebDriverWait wait;

    @BeforeAll
    static void setUp() {
        WebDriverManager.chromedriver().setup(); // Tự động tải và cấu hình driver

        // Disable JavaScript in Chrome
        ChromeOptions options = new ChromeOptions();
        Map<String, Object> prefs = new HashMap<>();
        prefs.put("profile.managed_default_content_settings.javascript", 2); // 1: Cho phép (default), 2: Chặn JavaScript
        options.setExperimentalOption("prefs", prefs);
        options.addArguments("--incognito"); // Ẩn danh

        driver = new ChromeDriver(options);
        wait = new WebDriverWait(driver, Duration.ofSeconds(10)); // Explicit Wait
        driver.manage().window().maximize();
    }

    @Test
    @Order(1)
    @DisplayName("Should login successfully with valid credentials")
    void testLoginSuccess() {
        driver.get("https://the-internet.herokuapp.com/login");

        driver.findElement(By.id("username")).sendKeys("tomsmith");
        driver.findElement(By.id("password")).sendKeys("SuperSecretPassword!");
        driver.findElement(By.cssSelector("button[type='submit']")).click();

        // Chờ thông báo thành công hiển thị
        WebElement successMsg = wait.until(
                ExpectedConditions.visibilityOfElementLocated(By.cssSelector(".flash.success"))
        );

        assertTrue(successMsg.getText().contains("You logged into a secure area!"));
    }

    @Test
    @Order(2)
    @DisplayName("Should display error when logging in with invalid credentials")
    void testLoginFail() {
        driver.get("https://the-internet.herokuapp.com/login");

        driver.findElement(By.id("username")).sendKeys("invalid");
        driver.findElement(By.id("password")).sendKeys("wrongpassword");
        driver.findElement(By.cssSelector("button[type='submit']")).click();

        // Chờ thông báo lỗi hiển thị
        WebElement errorMsg = wait.until(
                ExpectedConditions.visibilityOfElementLocated(By.cssSelector(".flash.error"))
        );

        assertTrue(errorMsg.getText().contains("Your username is invalid!"));
    }

    @AfterAll
    static void tearDown() {
        driver.quit();
    }
}
```

> Sau khi submit, **luôn chờ element hiển thị (wait)** rồi mới `.getText()` / `.click()`.

**Checklist:**

- [ ] Dùng `@TestMethodOrder`, `@Order`, `@DisplayName`
- [ ] Dùng Explicit Wait (`WebDriverWait` + `ExpectedConditions`), không dùng `Thread.sleep()`
- [ ] 2 test đều pass khi chạy
- [ ] `driver.quit()` trong `@AfterAll`

**Commit:**
```
test: add basic login success and failure tests for the-internet.herokuapp.com
```

---

## TODO 3 (Bài tập 1): Áp dụng `@ParameterizedTest` + `@CsvSource`

**Các bước:**

1. Thêm dependency vào `pom.xml`:

```xml
<!-- JUnit 5 Parameterized Test -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-params</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

2. Thêm imports vào `LoginTest.java`:

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
```

3. Thêm test `@Order(3)` vào `LoginTest.java`.

**Code đầy đủ – test method:**

```java
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
    driver.get("https://the-internet.herokuapp.com/login");

    driver.findElement(By.id("username")).sendKeys(username);
    driver.findElement(By.id("password")).sendKeys(password);
    driver.findElement(By.cssSelector("button[type='submit']")).click();

    By messageLocator = expectedResult.equals("success")
            ? By.cssSelector(".flash.success")
            : By.cssSelector(".flash.error");

    WebElement message = wait.until(ExpectedConditions.visibilityOfElementLocated(messageLocator));

    if (expectedResult.equals("success")) {
        assertTrue(message.getText().contains("You logged into a secure area!"));
    } else {
        assertTrue(message.getText().toLowerCase().contains("invalid"));
    }
}
```

**Checklist:**

- [ ] Dependency `junit-jupiter-params` đã thêm
- [ ] Test chạy đủ 4 bộ dữ liệu, tên test hiển thị theo `name = "..."`
- [ ] Cover cả positive lẫn negative case (ưu tiên negative test)

**Commit:**
```
test: add parameterized login test with @CsvSource for multiple credentials
```

---

## TODO 4 (Bài tập 2): Đọc dữ liệu test từ file CSV bên ngoài

**Các bước:**

1. Click phải thư mục `test` → `New → Directory` → chọn `resources`.
2. Trong `resources`: `New → File` → đặt tên `login-data.csv`.
3. Thêm test dùng `@CsvFileSource` (import `org.junit.jupiter.params.provider.CsvFileSource`).

**Code đầy đủ – `src/test/resources/login-data.csv`:**

```csv
username,password,expectedResult
tomsmith,SuperSecretPassword!,success
wronguser,SuperSecretPassword!,error
tomsmith,wrongpassword,error
, ,error
```

> Lưu ý: dùng `""` để biểu diễn chuỗi rỗng đúng cách (không phải khoảng trắng).

**Code đầy đủ – test method:**

```java
@ParameterizedTest(name = "Test login with: {0} / {1}")
@CsvFileSource(resources = "/login-data.csv", numLinesToSkip = 1)
@DisplayName("Login with data from external CSV file")
void testLoginWithCSV(String username, String password, String expectedResult) {
    driver.get("https://the-internet.herokuapp.com/login");

    // Chuyển null thành chuỗi rỗng + trim() để tránh lỗi khoảng trắng khi copy/paste CSV
    username = (username == null) ? "" : username.trim();
    password = (password == null) ? "" : password.trim();

    driver.findElement(By.id("username")).sendKeys(username);
    driver.findElement(By.id("password")).sendKeys(password);
    driver.findElement(By.cssSelector("button[type='submit']")).click();

    By messageLocator = expectedResult.equals("success")
            ? By.cssSelector(".flash.success")
            : By.cssSelector(".flash.error");

    WebElement message = wait.until(ExpectedConditions.visibilityOfElementLocated(messageLocator));

    if (expectedResult.equals("success")) {
        assertTrue(message.getText().contains("You logged into a secure area!"));
    } else {
        assertTrue(message.getText().toLowerCase().contains("invalid"));
    }
}
```

**Checklist:**

- [ ] `login-data.csv` nằm đúng trong `src/test/resources`
- [ ] Có `numLinesToSkip = 1` để bỏ dòng header
- [ ] Có xử lý `null` → `""` và `trim()`
- [ ] Test chạy đủ số dòng dữ liệu trong CSV

**Commit:**
```
test: add data-driven login test reading from external csv file
```

---

## TODO 5 (Bài tập 3): Tái cấu trúc theo Page Object Model – `Exercise3_POM`

**Các bước:**

1. Copy project trên → đổi tên thành `Exercise3_POM`.
2. Tạo 3 package trong `src/test/java`: `pages`, `tests`, `utils`.
3. Tạo `DriverFactory.java`, `LoginPage.java`, chuyển `LoginTest.java` sang dùng Page Object.
4. Chạy toàn bộ test, xác nhận pass.

**Code đầy đủ – `src/test/java/utils/DriverFactory.java`:**

```java
package utils;

import io.github.bonigarcia.wdm.WebDriverManager;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

import java.util.HashMap;
import java.util.Map;

public class DriverFactory {
    public static WebDriver createDriver() {
        WebDriverManager.chromedriver().setup();

        ChromeOptions options = new ChromeOptions();
        Map<String, Object> prefs = new HashMap<>();
        prefs.put("profile.managed_default_content_settings.javascript", 2);
        options.setExperimentalOption("prefs", prefs);
        options.addArguments("--incognito");

        return new ChromeDriver(options);
    }
}
```

**Code đầy đủ – `src/test/java/pages/LoginPage.java`:**

```java
package pages;

import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;

public class LoginPage {
    private WebDriver driver;

    // Constructor
    public LoginPage(WebDriver driver) {
        this.driver = driver;
    }

    // Locators
    private By usernameField = By.id("username");
    private By passwordField = By.id("password");
    private By loginButton = By.cssSelector("button[type='submit']");
    private By successMsg = By.cssSelector(".flash.success");
    private By errorMsg = By.cssSelector(".flash.error");

    // Actions
    public void navigate() {
        driver.get("https://the-internet.herokuapp.com/login");
    }

    public void login(String username, String password) {
        driver.findElement(usernameField).sendKeys(username);
        driver.findElement(passwordField).sendKeys(password);
        driver.findElement(loginButton).click();
    }

    public By getSuccessLocator() {
        return successMsg;
    }

    public By getErrorLocator() {
        return errorMsg;
    }
}
```

**Code đầy đủ – `src/test/java/tests/LoginTest.java`:**

```java
package tests;

import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvFileSource;
import org.junit.jupiter.params.provider.CsvSource;
import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.*;
import pages.LoginPage;
import utils.DriverFactory;

import java.time.Duration;

import static org.junit.jupiter.api.Assertions.assertTrue;

@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@DisplayName("Login Tests using Page Object Model")
public class LoginTest {
    static WebDriver driver;
    static WebDriverWait wait;
    static LoginPage loginPage;

    @BeforeAll
    static void setUp() {
        driver = DriverFactory.createDriver();
        wait = new WebDriverWait(driver, Duration.ofSeconds(10));
        driver.manage().window().maximize();
        loginPage = new LoginPage(driver);
    }

    @Test
    @Order(1)
    @DisplayName("Should login successfully with valid credentials")
    void testLoginSuccess() {
        loginPage.navigate();
        loginPage.login("tomsmith", "SuperSecretPassword!");
        WebElement success = wait.until(ExpectedConditions.visibilityOfElementLocated(loginPage.getSuccessLocator()));
        assertTrue(success.getText().contains("You logged into a secure area!"));
    }

    @Test
    @Order(2)
    @DisplayName("Should show error for invalid credentials")
    void testLoginFail() {
        loginPage.navigate();
        loginPage.login("wronguser", "wrongpassword");
        WebElement error = wait.until(ExpectedConditions.visibilityOfElementLocated(loginPage.getErrorLocator()));
        assertTrue(error.getText().toLowerCase().contains("invalid"));
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
        By resultLocator = expected.equals("success") ? loginPage.getSuccessLocator() : loginPage.getErrorLocator();
        WebElement result = wait.until(ExpectedConditions.visibilityOfElementLocated(resultLocator));

        if (expected.equals("success")) {
            assertTrue(result.getText().contains("You logged into a secure area!"));
        } else {
            assertTrue(result.getText().toLowerCase().contains("invalid"));
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
        By resultLocator = expected.equals("success") ? loginPage.getSuccessLocator() : loginPage.getErrorLocator();
        WebElement result = wait.until(ExpectedConditions.visibilityOfElementLocated(resultLocator));

        if (expected.equals("success")) {
            assertTrue(result.getText().contains("You logged into a secure area!"));
        } else {
            assertTrue(result.getText().toLowerCase().contains("invalid"));
        }
    }

    @AfterAll
    static void tearDown() {
        driver.quit();
    }
}
```

**Checklist:**

- [ ] Đúng cấu trúc `pages/`, `tests/`, `utils/`
- [ ] Test class không gọi trực tiếp `driver.findElement`
- [ ] Locators để `private` trong Page class
- [ ] Cả 4 test (Order 1→4) đều pass

**Commit (theo từng file):**
```
feat: add DriverFactory for shared webdriver configuration
feat: add LoginPage page object with locators and actions
refactor: migrate login tests to page object model
```

---

## TODO 6 (Bài tập 4): Xây dựng `BasePage` và `BaseTest` – `Exercise4_BasePage`

**Các bước:**

1. Copy project → đổi tên `Exercise4_BasePage`.
2. Tạo `pages/BasePage.java` – lớp cha chứa `driver`, `wait` và các thao tác an toàn.
3. Sửa `LoginPage extends BasePage`.
4. Tạo `tests/BaseTest.java` (abstract) quản lý vòng đời driver.
5. Sửa `LoginTest extends BaseTest`. Chạy lại toàn bộ test.

**Code đầy đủ – `src/test/java/pages/BasePage.java`:**

```java
package pages;

import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.*;

import java.time.Duration;

public class BasePage {
    protected WebDriver driver;
    protected WebDriverWait wait;

    public BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    // Wait for visibility
    protected WebElement waitForVisibility(By locator) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
    }

    // Click safely
    protected void click(By locator) {
        waitForVisibility(locator).click();
    }

    // Send keys safely
    protected void type(By locator, String text) {
        WebElement element = waitForVisibility(locator);
        element.clear();
        element.sendKeys(text);
    }

    // Get text safely
    protected String getText(By locator) {
        return waitForVisibility(locator).getText();
    }

    // Navigate to a URL
    public void navigateTo(String url) {
        driver.get(url);
    }

    // Check if element is present
    protected boolean isElementVisible(By locator) {
        try {
            return waitForVisibility(locator).isDisplayed();
        } catch (TimeoutException e) {
            return false;
        }
    }
}
```

**Code đầy đủ – `src/test/java/pages/LoginPage.java` (kế thừa BasePage):**

```java
package pages;

import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;

public class LoginPage extends BasePage {

    public LoginPage(WebDriver driver) {
        super(driver);
    }

    // Locators
    private By usernameField = By.id("username");
    private By passwordField = By.id("password");
    private By loginButton = By.cssSelector("button[type='submit']");
    private By successMsg = By.cssSelector(".flash.success");
    private By errorMsg = By.cssSelector(".flash.error");

    // Actions
    public void navigate() {
        navigateTo("https://the-internet.herokuapp.com/login");
    }

    public void login(String username, String password) {
        type(usernameField, username);
        type(passwordField, password);
        click(loginButton);
    }

    public By getSuccessLocator() {
        return successMsg;
    }

    public By getErrorLocator() {
        return errorMsg;
    }

    public String getMessageText(By locator) {
        return getText(locator);
    }
}
```

**Code đầy đủ – `src/test/java/tests/BaseTest.java`:**

```java
package tests;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.openqa.selenium.WebDriver;
import utils.DriverFactory;

public abstract class BaseTest {
    protected static WebDriver driver;

    @BeforeAll
    public static void setUpBase() {
        driver = DriverFactory.createDriver();
        driver.manage().window().maximize();
    }

    @AfterAll
    public static void tearDownBase() {
        if (driver != null) {
            driver.quit();
        }
    }
}
```

> Lưu ý: khai báo `abstract` cho `BaseTest` để tránh chạy nhầm test. Có thể mở rộng `@BeforeEach`/`@AfterEach` để ghi log mỗi test start.

**Code đầy đủ – `src/test/java/tests/LoginTest.java` (kế thừa BaseTest):**

```java
package tests;

import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvFileSource;
import org.junit.jupiter.params.provider.CsvSource;
import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.*;
import pages.LoginPage;

import java.time.Duration;

import static org.junit.jupiter.api.Assertions.assertTrue;

@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@DisplayName("Login Tests using Page Object Model")
public class LoginTest extends BaseTest {
    static WebDriverWait wait;
    static LoginPage loginPage;

    @BeforeAll
    static void initPage() {
        loginPage = new LoginPage(driver);
        wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    @Test
    @Order(1)
    @DisplayName("Should login successfully with valid credentials")
    void testLoginSuccess() {
        loginPage.navigate();
        loginPage.login("tomsmith", "SuperSecretPassword!");
        WebElement success = wait.until(ExpectedConditions.visibilityOfElementLocated(loginPage.getSuccessLocator()));
        assertTrue(success.getText().contains("You logged into a secure area!"));
    }

    @Test
    @Order(2)
    @DisplayName("Should show error for invalid credentials")
    void testLoginFail() {
        loginPage.navigate();
        loginPage.login("wronguser", "wrongpassword");
        WebElement error = wait.until(ExpectedConditions.visibilityOfElementLocated(loginPage.getErrorLocator()));
        assertTrue(error.getText().toLowerCase().contains("invalid"));
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
        By resultLocator = expected.equals("success") ? loginPage.getSuccessLocator() : loginPage.getErrorLocator();
        WebElement result = wait.until(ExpectedConditions.visibilityOfElementLocated(resultLocator));

        if (expected.equals("success")) {
            assertTrue(result.getText().contains("You logged into a secure area!"));
        } else {
            assertTrue(result.getText().toLowerCase().contains("invalid"));
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
        By resultLocator = expected.equals("success") ? loginPage.getSuccessLocator() : loginPage.getErrorLocator();
        WebElement result = wait.until(ExpectedConditions.visibilityOfElementLocated(resultLocator));

        if (expected.equals("success")) {
            assertTrue(result.getText().contains("You logged into a secure area!"));
        } else {
            assertTrue(result.getText().toLowerCase().contains("invalid"));
        }
    }
}
```

> Ghi chú: driver được quit ở `BaseTest.tearDownBase()` nên không cần `@AfterAll` riêng trong `LoginTest`.

**Checklist:**

- [ ] `BasePage` chứa đủ: waitForVisibility, click, type, getText, navigateTo, isElementVisible
- [ ] `LoginPage` kế thừa `BasePage`, không tự tạo wait riêng
- [ ] `BaseTest` là `abstract`, quản lý vòng đời driver
- [ ] `LoginTest` kế thừa `BaseTest`, tất cả test pass

**Commit (theo từng file):**
```
feat: add BasePage with reusable safe element interactions
refactor: make LoginPage extend BasePage
feat: add abstract BaseTest managing webdriver lifecycle
refactor: make LoginTest extend BaseTest
```

---

## Quy ước commit message (Conventional Commits)

Cú pháp: `<type>(scope tùy chọn): <mô tả ngắn, động từ nguyên mẫu, không viết hoa đầu, không dấu chấm cuối>`

| Type | Dùng khi |
|------|----------|
| `feat` | Thêm class/chức năng mới (Page Object, DriverFactory, BasePage...) |
| `test` | Thêm/sửa test case |
| `refactor` | Tái cấu trúc code không đổi hành vi (chuyển sang POM, kế thừa Base) |
| `chore` | Cấu hình project, pom.xml, dependency |
| `fix` | Sửa bug (locator sai, test flaky...) |
| `docs` | Tài liệu, README |

Luồng commit đầy đủ cho lab (TODO 1 → 6):

```
chore: init maven project with selenium, junit5 and webdrivermanager
test: add basic login success and failure tests for the-internet.herokuapp.com
test: add parameterized login test with @CsvSource for multiple credentials
test: add data-driven login test reading from external csv file
feat: add DriverFactory for shared webdriver configuration
feat: add LoginPage page object with locators and actions
refactor: migrate login tests to page object model
feat: add BasePage with reusable safe element interactions
refactor: make LoginPage extend BasePage
feat: add abstract BaseTest managing webdriver lifecycle
refactor: make LoginTest extend BaseTest
```

---

## Kiến thức cần nắm (tóm tắt lý thuyết)

- **Selenium WebDriver**: thư viện mã nguồn mở tự động hóa trình duyệt; hỗ trợ Java/Python/C#/JS và Chrome/Firefox/Edge...
- **Thành phần test script**: WebDriver/WebElement, Locator (`By.id`, `By.cssSelector`, `By.xpath`, `By.name`, `By.className`), Actions (`.click()`, `.sendKeys()`, `.getText()`), Waits.
- **Waits**: Implicit (toàn cục), **Explicit (`WebDriverWait` + `ExpectedConditions` – nên ưu tiên)**, Fluent (polling, xử lý exception).
- **ChromeOptions**: `--incognito` (ẩn danh), `--headless` (CI/CD), `prefs... javascript = 2` (chặn JS).
- **Test validation form**: luôn wait sau submit; assert bằng `assertTrue(msg.getText().contains(...))`; test trường trống, sai định dạng email/phone, password quá ngắn/dài; ưu tiên negative test.
