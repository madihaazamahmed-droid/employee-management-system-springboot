
# 1. Project structure

Create the project like this:

```text
employee_management_system/
│
├── src/
│   └── main/
│       ├── java/
│       │   └── com/
│       │       └── tcs/
│       │           └── ems/
│       │               │
│       │               ├── EmsApplication.java
│       │               │
│       │               ├── config/
│       │               │   └── SecurityConfig.java
│       │               │
│       │               ├── controller/
│       │               │   ├── EmployeeController.java
│       │               │   └── UserController.java
│       │               │
│       │               ├── dto/
│       │               │   ├── RegisterRequest.java
│       │               │   └── VerifyOtpRequest.java
│       │               │
│       │               ├── entity/
│       │               │   ├── Employee.java
│       │               │   └── User.java
│       │               │
│       │               ├── exception/
│       │               │   ├── GlobalExceptionHandling.java
│       │               │   ├── InvalidOtpException.java
│       │               │   ├── OtpAlreadyVerified.java
│       │               │   ├── OtpExpiredException.java
│       │               │   ├── OtpVerifiedException.java
│       │               │   └── UserNotFoundException.java
│       │               │
│       │               ├── repository/
│       │               │   ├── EmployeeRepository.java
│       │               │   └── UserRepository.java
│       │               │
│       │               ├── service/
│       │               │   ├── EmployeeService.java
│       │               │   ├── OtpService.java
│       │               │   └── UserService.java
│       │               │
│       │               └── util/
│       │                   ├── EmailService.java
│       │                   └── OtpGenerator.java
│       │
│       └── resources/
│           └── application.properties
│
├── .gitignore
├── pom.xml
└── README.md
```

---

# 2. `EmsApplication.java`

```java
package com.tcs.ems;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@SpringBootApplication
public class EmsApplication {

    public static void main(String[] args) {
        SpringApplication.run(EmsApplication.class, args);
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

# 3. `config/SecurityConfig.java`

```java
package com.tcs.ems.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth

                .requestMatchers(
                    "/users/register",
                    "/users/verify-otp"
                ).permitAll()

                .requestMatchers(
                    HttpMethod.GET,
                    "/employees/**"
                ).hasAnyRole("ADMIN", "USER")

                .requestMatchers(
                    "/employees/**"
                ).hasRole("ADMIN")

                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }

    @Bean
    UserDetailsService userDetailsService(
            PasswordEncoder passwordEncoder) {

        UserDetails admin = User
                .withUsername("admin")
                .password(passwordEncoder.encode("admin123"))
                .roles("ADMIN")
                .build();

        UserDetails user = User
                .withUsername("user")
                .password(passwordEncoder.encode("user123"))
                .roles("USER")
                .build();

        return new InMemoryUserDetailsManager(admin, user);
    }
}
```

> Your current project uses **HTTP Basic + in-memory ADMIN/USER authentication**. It is not actually JWT authentication yet, so I would **not claim JWT in the README** unless you add the JWT implementation.

---

# 4. `controller/EmployeeController.java`

I recommend fixing your current create method because right now you commented out the actual service call.

```java
package com.tcs.ems.controller;

import java.util.List;
import java.util.Optional;

import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import com.tcs.ems.entity.Employee;
import com.tcs.ems.service.EmployeeService;

import jakarta.validation.Valid;

@RestController
@RequestMapping("/employees")
public class EmployeeController {

    private final EmployeeService employeeService;

    public EmployeeController(EmployeeService employeeService) {
        this.employeeService = employeeService;
    }

    @PostMapping
    public String createEmployee(
            @Valid @RequestBody Employee employee) {

        return employeeService.createEmployee(employee);
    }

    @GetMapping("/{id}")
    public Optional<Employee> fetchEmployeeById(
            @PathVariable Long id) {

        return employeeService.fetchEmployeeById(id);
    }

    @GetMapping
    public List<Employee> fetchAllEmployee() {

        return employeeService.fetchAllEmployee();
    }

    @DeleteMapping("/{id}")
    public String deleteEmployeeById(
            @PathVariable Long id) {

        return employeeService.deleteEmployeeById(id);
    }

    @PutMapping("/{id}")
    public String updateEmployeeById(
            @PathVariable Long id,
            @Valid @RequestBody Employee employee) {

        return employeeService.updateEmployeeById(id, employee);
    }
}
```

---

# 5. `controller/UserController.java`

Your current controller creates a `UserService` parameter but doesn't store it. Fix that.

```java
package com.tcs.ems.controller;

import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import com.tcs.ems.dto.RegisterRequest;
import com.tcs.ems.dto.VerifyOtpRequest;
import com.tcs.ems.service.OtpService;
import com.tcs.ems.service.UserService;

import jakarta.validation.Valid;

@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;
    private final OtpService otpService;

    public UserController(
            UserService userService,
            OtpService otpService) {

        this.userService = userService;
        this.otpService = otpService;
    }

    @PostMapping("/register")
    public String register(
            @Valid @RequestBody RegisterRequest registerRequest) {

        return userService.register(registerRequest);
    }

    @PostMapping("/verify-otp")
    public String verifyOtp(
            @Valid @RequestBody VerifyOtpRequest verifyOtpRequest) {

        return otpService.verifyOtp(verifyOtpRequest);
    }
}
```

---

# 6. `dto/RegisterRequest.java`

```java
package com.tcs.ems.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.Data;

@Data
public class RegisterRequest {

    @NotBlank(message = "Name is required")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Enter a valid email")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 6, message = "Password must contain at least 6 characters")
    private String password;
}
```

---

# 7. `dto/VerifyOtpRequest.java`

```java
package com.tcs.ems.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import lombok.Data;

@Data
public class VerifyOtpRequest {

    @NotBlank(message = "Email is required")
    @Email(message = "Enter a valid email")
    private String email;

    @NotBlank(message = "OTP is required")
    private String otp;
}
```

---

# 8. `entity/Employee.java`

Your current Employee uses `email` as the primary key while the repository uses `Long`. That's a mismatch.

For a proper EMS, use an `id`.

```java
package com.tcs.ems.entity;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.PositiveOrZero;
import lombok.Data;

@Entity
@Data
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank(message = "Name is required")
    private String name;

    @Email(message = "Enter a valid email id")
    @NotBlank(message = "Email cannot be empty")
    private String email;

    @PositiveOrZero(message = "Salary cannot be negative")
    private double salary;

    @NotBlank(message = "Department is required")
    private String department;
}
```

---

# 9. `entity/User.java`

```java
package com.tcs.ems.entity;

import java.time.LocalDateTime;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.Data;

@Entity
@Data
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @NotBlank(message = "Name is required")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Enter a valid email")
    @Column(unique = true)
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 6, message = "Password must contain at least 6 characters")
    private String password;

    private String role;

    private boolean verified;

    private String otp;

    private LocalDateTime otpexpirytime;
}
```

---

# 10. `exception/InvalidOtpException.java`

```java
package com.tcs.ems.exception;

public class InvalidOtpException extends RuntimeException {

    public InvalidOtpException(String message) {
        super(message);
    }
}
```

---

# 11. `exception/OtpAlreadyVerified.java`

```java
package com.tcs.ems.exception;

public class OtpAlreadyVerified extends RuntimeException {

    public OtpAlreadyVerified(String message) {
        super(message);
    }
}
```

---

# 12. `exception/OtpExpiredException.java`

```java
package com.tcs.ems.exception;

public class OtpExpiredException extends RuntimeException {

    public OtpExpiredException(String message) {
        super(message);
    }
}
```

---

# 13. `exception/OtpVerifiedException.java`

You were using this class but hadn't included its code.

```java
package com.tcs.ems.exception;

public class OtpVerifiedException extends RuntimeException {

    public OtpVerifiedException(String message) {
        super(message);
    }
}
```

---

# 14. `exception/UserNotFoundException.java`

```java
package com.tcs.ems.exception;

public class UserNotFoundException extends RuntimeException {

    public UserNotFoundException(String message) {
        super(message);
    }
}
```

---

# 15. `exception/GlobalExceptionHandling.java`

```java
package com.tcs.ems.exception;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandling {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> invalidData(
            MethodArgumentNotValidException exception) {

        Map<String, String> message = new HashMap<>();

        List<FieldError> errors =
                exception.getBindingResult().getFieldErrors();

        for (FieldError fe : errors) {
            message.put(fe.getField(), fe.getDefaultMessage());
        }

        return new ResponseEntity<>(
                message,
                HttpStatus.BAD_REQUEST
        );
    }

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<String> userNotFound(
            UserNotFoundException exception) {

        return new ResponseEntity<>(
                exception.getMessage(),
                HttpStatus.NOT_FOUND
        );
    }

    @ExceptionHandler(InvalidOtpException.class)
    public ResponseEntity<String> invalidOtp(
            InvalidOtpException exception) {

        return new ResponseEntity<>(
                exception.getMessage(),
                HttpStatus.BAD_REQUEST
        );
    }

    @ExceptionHandler(OtpVerifiedException.class)
    public ResponseEntity<String> otpVerified(
            OtpVerifiedException exception) {

        return new ResponseEntity<>(
                exception.getMessage(),
                HttpStatus.OK
        );
    }

    @ExceptionHandler(OtpExpiredException.class)
    public ResponseEntity<String> otpExpired(
            OtpExpiredException exception) {

        return new ResponseEntity<>(
                exception.getMessage(),
                HttpStatus.GONE
        );
    }

    @ExceptionHandler(OtpAlreadyVerified.class)
    public ResponseEntity<String> otpAlreadyVerified(
            OtpAlreadyVerified exception) {

        return new ResponseEntity<>(
                exception.getMessage(),
                HttpStatus.ACCEPTED
        );
    }
}
```

---

# 16. `repository/EmployeeRepository.java`

```java
package com.tcs.ems.repository;

import org.springframework.data.jpa.repository.JpaRepository;

import com.tcs.ems.entity.Employee;

public interface EmployeeRepository
        extends JpaRepository<Employee, Long> {

}
```

---

# 17. `repository/UserRepository.java`

```java
package com.tcs.ems.repository;

import java.util.Optional;

import org.springframework.data.jpa.repository.JpaRepository;

import com.tcs.ems.entity.User;

public interface UserRepository
        extends JpaRepository<User, Integer> {

    Optional<User> findByEmail(String email);
}
```

---

# 18. `service/EmployeeService.java`

```java
package com.tcs.ems.service;

import java.util.List;
import java.util.Optional;

import org.springframework.stereotype.Service;

import com.tcs.ems.entity.Employee;
import com.tcs.ems.repository.EmployeeRepository;

@Service
public class EmployeeService {

    private final EmployeeRepository employeeRepository;

    public EmployeeService(
            EmployeeRepository employeeRepository) {

        this.employeeRepository = employeeRepository;
    }

    public String createEmployee(Employee employee) {

        employeeRepository.save(employee);

        return "Employee data inserted";
    }

    public Optional<Employee> fetchEmployeeById(Long id) {

        return employeeRepository.findById(id);
    }

    public List<Employee> fetchAllEmployee() {

        return employeeRepository.findAll();
    }

    public String deleteEmployeeById(Long id) {

        Optional<Employee> employee =
                employeeRepository.findById(id);

        if (employee.isPresent()) {

            employeeRepository.deleteById(id);

            return "Employee data deleted";
        }

        return "Employee not found";
    }

    public String updateEmployeeById(
            Long id,
            Employee employee) {

        Optional<Employee> existingEmployee =
                employeeRepository.findById(id);

        if (existingEmployee.isPresent()) {

            Employee existing = existingEmployee.get();

            existing.setName(employee.getName());
            existing.setEmail(employee.getEmail());
            existing.setDepartment(employee.getDepartment());
            existing.setSalary(employee.getSalary());

            employeeRepository.save(existing);

            return "Employee data updated";
        }

        return "Employee not found";
    }
}
```

---

# 19. `service/OtpService.java`

```java
package com.tcs.ems.service;

import java.time.LocalDateTime;
import java.util.Optional;

import org.springframework.stereotype.Service;

import com.tcs.ems.dto.VerifyOtpRequest;
import com.tcs.ems.entity.User;
import com.tcs.ems.exception.InvalidOtpException;
import com.tcs.ems.exception.OtpAlreadyVerified;
import com.tcs.ems.exception.OtpExpiredException;
import com.tcs.ems.exception.OtpVerifiedException;
import com.tcs.ems.exception.UserNotFoundException;
import com.tcs.ems.repository.UserRepository;

@Service
public class OtpService {

    private final UserRepository userRepository;

    public OtpService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public String verifyOtp(
            VerifyOtpRequest verifyOtpRequest) {

        Optional<User> optionalUser =
                userRepository.findByEmail(
                        verifyOtpRequest.getEmail());

        if (optionalUser.isEmpty()) {
            throw new UserNotFoundException(
                    "No user found");
        }

        User user = optionalUser.get();

        if (user.isVerified()) {
            throw new OtpAlreadyVerified(
                    "OTP Already Verified");
        }

        if (user.getOtp() == null) {
            throw new InvalidOtpException(
                    "Invalid OTP");
        }

        if (!user.getOtp().equals(
                verifyOtpRequest.getOtp())) {

            throw new InvalidOtpException(
                    "Invalid OTP");
        }

        if (user.getOtpexpirytime() == null ||
                LocalDateTime.now().isAfter(
                        user.getOtpexpirytime())) {

            throw new OtpExpiredException(
                    "OTP Expired");
        }

        user.setVerified(true);
        user.setOtp(null);
        user.setOtpexpirytime(null);

        userRepository.save(user);

        throw new OtpVerifiedException(
                "OTP verified successfully");
    }
}
```

---

# 20. `service/UserService.java`

```java
package com.tcs.ems.service;

import java.time.LocalDateTime;
import java.util.Optional;

import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;

import com.tcs.ems.dto.RegisterRequest;
import com.tcs.ems.entity.User;
import com.tcs.ems.repository.UserRepository;
import com.tcs.ems.util.EmailService;
import com.tcs.ems.util.OtpGenerator;

@Service
public class UserService {

    private final UserRepository userRepository;
    private final EmailService emailService;
    private final PasswordEncoder passwordEncoder;

    public UserService(
            UserRepository userRepository,
            EmailService emailService,
            PasswordEncoder passwordEncoder) {

        this.userRepository = userRepository;
        this.emailService = emailService;
        this.passwordEncoder = passwordEncoder;
    }

    public String register(
            RegisterRequest registerRequest) {

        Optional<User> existingUser =
                userRepository.findByEmail(
                        registerRequest.getEmail());

        if (existingUser.isPresent()) {
            return "Email ID already exists";
        }

        User user = new User();

        user.setName(registerRequest.getName());
        user.setEmail(registerRequest.getEmail());

        user.setPassword(
                passwordEncoder.encode(
                        registerRequest.getPassword()));

        user.setRole("USER");
        user.setVerified(false);

        String otp = OtpGenerator.generateOtp();

        user.setOtp(otp);
        user.setOtpexpirytime(
                LocalDateTime.now().plusMinutes(5));

        userRepository.save(user);

        emailService.sendOtp(
                registerRequest.getEmail(),
                otp);

        return "Please check your email for OTP";
    }
}
```

---

# 21. `util/EmailService.java`

```java
package com.tcs.ems.util;

import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.stereotype.Service;

@Service
public class EmailService {

    private final JavaMailSender javaMailSender;

    public EmailService(
            JavaMailSender javaMailSender) {

        this.javaMailSender = javaMailSender;
    }

    public void sendOtp(
            String toEmail,
            String otp) {

        SimpleMailMessage message =
                new SimpleMailMessage();

        message.setTo(toEmail);
        message.setSubject("OTP Verification");
        message.setText(
                "Your OTP is: " + otp);

        javaMailSender.send(message);
    }
}
```

---

# 22. `util/OtpGenerator.java`

```java
package com.tcs.ems.util;

import java.util.Random;

public class OtpGenerator {

    private OtpGenerator() {
    }

    public static String generateOtp() {

        Random random = new Random();

        int otp =
                100000 + random.nextInt(900000);

        return String.valueOf(otp);
    }
}
```

---

# 23. `application.properties`

**Do NOT put your real Gmail password here.**

```properties
spring.application.name=employee-management-system

server.port=8083

# MySQL
spring.datasource.url=jdbc:mysql://localhost:3306/userdb?createDatabaseIfNotExist=true
spring.datasource.username=root
spring.datasource.password=YOUR_MYSQL_PASSWORD

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Gmail SMTP
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=${MAIL_USERNAME}
spring.mail.password=${MAIL_PASSWORD}

spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

Then run with environment variables:

```bash
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-app-password
```

On Windows, you can configure them through Environment Variables or your IDE's run configuration.

---

# 24. `.gitignore`

Create this in the project root:

```gitignore
target/

.mvn/

.idea/
*.iml

.classpath
.project
.settings/

.vscode/

*.log

.env
.env.*

application-local.properties
application-secret.properties

.DS_Store
```

---

# 25. `pom.xml`

I would simplify your current POM rather than saving all the generated `*-test` dependencies.

```xml
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
         http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.1.0</version>
        <relativePath/>
    </parent>

    <groupId>com.tcs</groupId>
    <artifactId>employee-management-system</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <name>Employee Management System</name>
    <description>
        Employee Management System using Spring Boot
    </description>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>

        <!-- Spring Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webmvc</artifactId>
        </dependency>

        <!-- Spring Data JPA -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Spring Security -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <!-- Email -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-mail</artifactId>
        </dependency>

        <!-- MySQL -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- DevTools -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>

        <!-- Tests -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <build>
        <plugins>

            <plugin>
                <groupId>
                    org.springframework.boot
                </groupId>

                <artifactId>
                    spring-boot-maven-plugin
                </artifactId>

                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>
                                org.projectlombok
                            </groupId>

                            <artifactId>
                                lombok
                            </artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>

        </plugins>
    </build>

</project>
```

---

# 26. README.md

For this **reference repository**, don't call it Day 11.

Use:

````markdown
# Employee Management System

A complete Employee Management System built using Java and Spring Boot.

This project demonstrates user registration, email OTP verification,
password encryption, employee CRUD operations, Spring Security,
role-based authorization, Bean Validation and Global Exception Handling.

---

## Technologies

- Java 21
- Spring Boot
- Spring MVC
- Spring Data JPA
- Hibernate
- Spring Security
- MySQL
- Jakarta Bean Validation
- Maven
- Lombok
- JavaMail
- Git
- GitHub
- Postman

---

## Project Architecture

```text
Client
   |
   v
Controller
   |
   v
Validation
   |
   v
Service
   |
   v
Repository
   |
   v
MySQL
````

Exception flow:

```text
Exception
   |
   v
@RestControllerAdvice
   |
   v
Exception Handler
   |
   v
HTTP Response
```

---

## Project Structure

```text
com.tcs.ems
│
├── config
│   └── SecurityConfig.java
│
├── controller
│   ├── EmployeeController.java
│   └── UserController.java
│
├── dto
│   ├── RegisterRequest.java
│   └── VerifyOtpRequest.java
│
├── entity
│   ├── Employee.java
│   └── User.java
│
├── exception
│   ├── GlobalExceptionHandling.java
│   ├── InvalidOtpException.java
│   ├── OtpAlreadyVerified.java
│   ├── OtpExpiredException.java
│   ├── OtpVerifiedException.java
│   └── UserNotFoundException.java
│
├── repository
│   ├── EmployeeRepository.java
│   └── UserRepository.java
│
├── service
│   ├── EmployeeService.java
│   ├── OtpService.java
│   └── UserService.java
│
└── util
    ├── EmailService.java
    └── OtpGenerator.java
```

---

# Features

## 1. User Registration

A new user can register using:

* Name
* Email
* Password

The password is encrypted using BCrypt before being stored.

---

## 2. Email OTP Verification

After registration:

```text
Register
   |
   v
Generate OTP
   |
   v
Save OTP
   |
   v
Send OTP through Email
   |
   v
Verify OTP
   |
   v
User Account Verified
```

The OTP expires after 5 minutes.

---

## 3. Employee CRUD

The application provides:

```text
POST   /employees
GET    /employees
GET    /employees/{id}
PUT    /employees/{id}
DELETE /employees/{id}
```

---

## 4. Bean Validation

The project uses Jakarta Bean Validation.

Examples:

```java
@NotBlank
@Email
@NotNull
@Size
@PositiveOrZero
```

Invalid requests are rejected before reaching the service layer.

---

## 5. Global Exception Handling

The project uses:

```java
@RestControllerAdvice
```

Validation errors are handled using:

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
```

Example response:

```json
{
    "name": "Name is required",
    "email": "Enter a valid email"
}
```

Custom exceptions include:

```text
UserNotFoundException
InvalidOtpException
OtpExpiredException
OtpAlreadyVerified
OtpVerifiedException
```

---

## 6. Spring Security

The application uses Spring Security with HTTP Basic authentication.

Configured users:

```text
ADMIN
Username: admin
Password: admin123

USER
Username: user
Password: user123
```

These credentials are for local learning/testing only.

---

## Role-Based Authorization

### ADMIN

Can:

```text
GET employees
POST employees
PUT employees
DELETE employees
```

### USER

Can:

```text
GET employees
```

The employee endpoints are protected using Spring Security.

---

# API Endpoints

| Method | Endpoint            | Description       |
| ------ | ------------------- | ----------------- |
| POST   | `/users/register`   | Register user     |
| POST   | `/users/verify-otp` | Verify OTP        |
| POST   | `/employees`        | Create employee   |
| GET    | `/employees`        | Get all employees |
| GET    | `/employees/{id}`   | Get employee      |
| PUT    | `/employees/{id}`   | Update employee   |
| DELETE | `/employees/{id}`   | Delete employee   |

---

# Registration Request

```json
{
    "name": "Madiha",
    "email": "madiha@example.com",
    "password": "password123"
}
```

---

# OTP Verification Request

```json
{
    "email": "madiha@example.com",
    "otp": "123456"
}
```

---

# Employee Request

```json
{
    "name": "John",
    "email": "john@example.com",
    "salary": 50000,
    "department": "IT"
}
```

---

# Database

MySQL is used as the database.

Database:

```text
userdb
```

Tables are automatically generated by Hibernate.

Main tables:

```text
users
employees
```

---

# Configuration

Configure MySQL in:

```text
src/main/resources/application.properties
```

Example:

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/userdb
spring.datasource.username=root
spring.datasource.password=YOUR_PASSWORD
```

Email credentials should be provided through environment variables.

Never commit passwords or API credentials to GitHub.

---

# Running the Project

Clone:

```bash
git clone https://github.com/madihaazamahmed-droid/employee_management_system.git
```

Enter the project:

```bash
cd employee_management_system
```

Build:

```bash
mvn clean install
```

Run:

```bash
mvn spring-boot:run
```

Application:

```text
http://localhost:8083
```

---

# Postman Testing Flow

```text
1. Register User
       |
       v
2. Receive OTP
       |
       v
3. Verify OTP
       |
       v
4. Authenticate
       |
       v
5. Access Employee APIs
```

---

# Concepts Learned

* Java
* OOP
* Spring Boot
* REST APIs
* Dependency Injection
* Spring MVC
* Spring Data JPA
* Hibernate
* MySQL
* DTO
* Repository Pattern
* Service Layer
* Controller Layer
* CRUD Operations
* Email Integration
* OTP Verification
* Password Encryption
* Spring Security
* Role-Based Authorization
* Bean Validation
* Custom Exceptions
* Global Exception Handling
* HTTP Status Codes
* Maven
* Git
* GitHub
* Postman

---

# Future Enhancements

* JWT Authentication
* Refresh Tokens
* Pagination
* Sorting
* Search
* Swagger/OpenAPI
* Unit Testing
* Integration Testing
* React Frontend
* Docker
* CI/CD
* Cloud Deployment

---

# Author

Madiha Azam Ahmed

GitHub:

[https://github.com/madihaazamahmed-droid](https://github.com/madihaazamahmed-droid)

---
