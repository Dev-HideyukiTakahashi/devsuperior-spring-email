# 📧 Envio de E-mail via Gmail com Spring Boot

## 🔑 Criando uma senha de aplicativo no Google

Para enviar e-mails usando o Gmail, é necessário gerar uma senha de aplicativo na sua conta do Google. Siga este guia:
🔗 [Como criar uma senha de app](https://support.google.com/accounts/answer/185833)

### Passos:

1. Acesse sua conta do Google: [myaccount.google.com](https://myaccount.google.com/)
2. Vá para **Segurança** > **Validação em duas etapas** > **Senhas de app**
3. Selecione **Outro**, dê um nome ao app e clique em **Gerar**
4. Existe a opção de digitar **senhas app** na busca para ser direcionado a página


## ⚙ Configuração no Spring Boot

### 📄 `application.properties`

```properties
spring.mail.host=${EMAIL_HOST:smtp.gmail.com}
spring.mail.port=${EMAIL_PORT:587}
spring.mail.username=${EMAIL_USERNAME:test@gmail.com}
spring.mail.password=${EMAIL_PASSWORD:123456}
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

### 📦 Dependência no `pom.xml`

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

## ✉ Criando DTO para E-mail

```java
public class EmailDTO {
    @NotBlank
    @Email
    private String to;

    @NotBlank
    private String subject;

    @NotBlank
    private String body;

    // Construtor, Getters e Setters
}
```

## 🛠 Implementação do Service de E-mail

Criação de uma **Exception personalizada** para erros no envio de e-mails:

```java
@Service
public class EmailService {

    @Value("${spring.mail.username}")
    private String emailFrom;

    @Autowired
    private JavaMailSender emailSender;

    public void sendEmail(EmailDTO obj) {
        try {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setFrom(emailFrom);
            message.setTo(obj.getTo());
            message.setSubject(obj.getSubject());
            message.setText(obj.getBody());
            emailSender.send(message);
        } catch (MailException e) {
            throw new EmailException("Falha ao enviar e-mail");
        }
    }
}
```

---

# 🔐 Caso de Uso: Recuperação de Senha

### 📜 Cenário Principal

1️⃣ [IN] O usuário informa seu e-mail
2️⃣ [OUT] O sistema gera um **token de recuperação** e informa sua validade
3️⃣ [IN] O usuário informa o token e define uma **nova senha**

### ⚠ Possíveis Erros

❌ **Email inválido** → O sistema informa que o e-mail é inválido
❌ **Email não encontrado** → O sistema informa que o e-mail não foi localizado
❌ **Token inválido** → O sistema informa que o token de recuperação não é válido
❌ **Erro de validação** → O sistema informa que a senha não atende os critérios

📌 **Critérios para senha válida:** Mínimo de **8 caracteres**

### 📄 `application.properties`

```properties
email.password-recover.token.minutes=${PASSWORD_RECOVER_TOKEN_MINUTES:30}
email.password-recover.uri=${PASSWORD_RECOVER_URI:http://localhost:5173/recover-password/}
```


## 🏛 Criando a Entidade para Recuperação de Senha

```java
@Entity
public class PasswordRecover {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String token;

    @Column(nullable = false)
    private String email;

    @Column(nullable = false)
    private Instant expiration;

    // Construtor, Getters e Setters
}
```

## 📌 DTOs

### `NewPasswordDTO`

```java
public class NewPasswordDTO {
    @NotBlank(message = "Campo obrigatório")
    private String token;

    @NotBlank(message = "Campo obrigatório")
    @Size(min = 8, message = "Deve ter no mínimo 8 caracteres")
    private String password;

    // Getters e Setters
}
```

### `EmailMinDTO`

```java
public class EmailMinDTO {
    @NotBlank(message = "Campo obrigatório")
    @Email
    private String email;

    // Getters e Setters
}
```


## 🌍 Implementação do Controller

```java
@RestController
@RequestMapping("/auth")
public class AuthController {

    private final AuthService authService;

    public AuthController(AuthService authService) {
        this.authService = authService;
    }

    @PostMapping("/recover-token")
    public ResponseEntity<Void> createRecoverToken(@Valid @RequestBody EmailMinDTO dto) {
        authService.createRecoverToken(dto);
        return ResponseEntity.noContent().build();
    }

    @PutMapping("/new-password")
    public ResponseEntity<Void> saveNewPassword(@Valid @RequestBody NewPasswordDTO dto) {
        authService.saveNewPassword(dto);
        return ResponseEntity.noContent().build();
    }
}
```

## 🛠 Implementação do Service

```java
@Service
public class AuthService {

    @Value("${email.password-recover.token.minutes}")
    private Long tokenMinutes;

    @Value("${email.password-recover.uri}")
    private String recoverUri;

    private final UserRepository userRepository;
    private final PasswordRecoverRepository passwordRecoverRepository;
    private final EmailService emailService;
    private final PasswordEncoder passwordEncoder;

    public AuthService(UserRepository userRepository, PasswordRecoverRepository passwordRecoverRepository, EmailService emailService, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordRecoverRepository = passwordRecoverRepository;
        this.emailService = emailService;
        this.passwordEncoder = passwordEncoder;
    }

    @Transactional
    public void createRecoverToken(EmailMinDTO dto) {
        User user = userRepository.findByEmail(dto.getEmail());
        if (user == null) {
            throw new ResourceNotFoundException("Email não encontrado");
        }

        String token = UUID.randomUUID().toString();
        PasswordRecover entity = new PasswordRecover();
        entity.setEmail(dto.getEmail());
        entity.setToken(token);
        entity.setExpiration(Instant.now().plusSeconds(tokenMinutes * 60L));
        passwordRecoverRepository.save(entity);

        String body = "Acesse o link para definir uma nova senha:\n" + recoverUri + token + "\nValidade de " + tokenMinutes + " minutos";
        emailService.sendEmail(new EmailDTO(dto.getEmail(), "Recuperação de Senha", body));
    }
}
```

## 🔍 Implementação do Repositório

```java
@Query("SELECT obj FROM PasswordRecover obj WHERE obj.token = :token AND obj.expiration > :now")
List<PasswordRecover> searchValidTokens(String token, Instant now);
```
