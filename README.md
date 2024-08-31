public public static void main(String[] args) {
    // Classe Cliente
    @Entity
    public class Cliente {
        @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @NotBlank
        @Pattern(regexp = "\\d{11}", message = "CPF deve conter 11 dígitos")
        private String cpf;

        @NotBlank
        private String nome;

        @Email
        private String email;

        private Double saldo = 0.0;

        // getters e setters
    }

// Classe Empresa
    @Entity
    public class Empresa {
        @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @NotBlank
        @Pattern(regexp = "\\d{14}", message = "CNPJ deve conter 14 dígitos")
        private String cnpj;

        @NotBlank
        private String nome;

        private Double saldo = 0.0;

        @NotNull
        private Double taxaSistema;

        // getters e setters
    }

// Classe Transacao
    @Entity
    public class Transacao {
        @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @ManyToOne
        private Cliente cliente;

        @ManyToOne
        private Empresa empresa;

        private Double valor;
        private LocalDateTime dataHora;
        private String tipo; // "DEPOSITO" ou "SAQUE"

        // getters e setters
    }
    @Pattern(regexp = "\\d{11}", message = "CPF deve conter 11 dígitos")
    @Pattern(regexp = "\\d{14}", message = "CNPJ deve conter 14 dígitos")
    @Service
    public class TransacaoService {

        @Autowired
        private ClienteRepository clienteRepository;

        @Autowired
        private EmpresaRepository empresaRepository;

        @Autowired
        private TransacaoRepository transacaoRepository;

        @Autowired
        private EmailService emailService;

        public void realizarDeposito(Long clienteId, Long empresaId, Double valor) {
            Cliente cliente = clienteRepository.findById(clienteId).orElseThrow();
            Empresa empresa = empresaRepository.findById(empresaId).orElseThrow();

            double valorFinal = valor - empresa.getTaxaSistema();
            empresa.setSaldo(empresa.getSaldo() + valorFinal);
            cliente.setSaldo(cliente.getSaldo() - valor);

            clienteRepository.save(cliente);
            empresaRepository.save(empresa);

            Transacao transacao = new Transacao(cliente, empresa, valor, LocalDateTime.now(), "DEPOSITO");
            transacaoRepository.save(transacao);

            enviarNotificacao(cliente, empresa, "Depósito realizado com sucesso.");
            enviarCallback(empresa);
        }

        public void realizarSaque(Long clienteId, Long empresaId, Double valor) {
            Cliente cliente = clienteRepository.findById(clienteId).orElseThrow();
            Empresa empresa = empresaRepository.findById(empresaId).orElseThrow();

            double valorFinal = valor + empresa.getTaxaSistema();
            if (empresa.getSaldo() >= valorFinal) {
                empresa.setSaldo(empresa.getSaldo() - valorFinal);
                cliente.setSaldo(cliente.getSaldo() + valor);

                clienteRepository.save(cliente);
                empresaRepository.save(empresa);

                Transacao transacao = new Transacao(cliente, empresa, valor, LocalDateTime.now(), "SAQUE");
                transacaoRepository.save(transacao);

                enviarNotificacao(cliente, empresa, "Saque realizado com sucesso.");
                enviarCallback(empresa);
            } else {
                throw new RuntimeException("Saldo insuficiente na empresa.");
            }
        }

        private void enviarNotificacao(Cliente cliente, Empresa empresa, String mensagem) {
            emailService.enviarEmail(cliente.getEmail(), "Transação na empresa " + empresa.getNome(), mensagem);
        }

        private void enviarCallback(Empresa empresa) {
            RestTemplate restTemplate = new RestTemplate();
            String CALLBACK_URL = "https://webhook.site/your-webhook-url";
            try {
                restTemplate.postForLocation(CALLBACK_URL, empresa);
            } catch (Exception e) {
                System.err.println("Erro ao enviar callback: " + e.getMessage());
            }
        }
    }
    @RestControllerAdvice
    public class GlobalExceptionHandler {

        @ExceptionHandler(RuntimeException.class)
        public ResponseEntity<String> handleRuntimeException(RuntimeException ex) {
            return ResponseEntity.badRequest().body(ex.getMessage());
        }
    }
    @Service
    public class EmailService {

        @Autowired
        private JavaMailSender emailSender;

        public void enviarEmail(String para, String assunto, String texto) {
            SimpleMailMessage mensagem = new SimpleMailMessage();
            mensagem.setTo(para);
            mensagem.setSubject(assunto);
            mensagem.setText(texto);
            emailSender.send(mensagem);
        }
    }
    @SpringBootTest
    public class TransacaoServiceTest {

        @Autowired
        private TransacaoService transacaoService;

        @MockBean
        private ClienteRepository clienteRepository;

        @MockBean
        private EmpresaRepository empresaRepository;

        @Test
        void realizarDepositoTest() {
            // Testes aqui
        }
    }
    @SpringBootApplication
    public class FinanceiroApplication {
        public static void main(String[] args) {
            SpringApplication.run(FinanceiroApplication.class, args);
        }
    }

}
