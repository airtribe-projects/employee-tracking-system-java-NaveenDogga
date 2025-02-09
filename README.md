#application.yml

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/employee_db
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true

  cache:
    type: redis
  redis:
    host: localhost
    port: 6379

  security:
    oauth2:
      client:
        registration:
          google:
            client-id: your-client-id
            client-secret: your-client-secret
            scope: profile, email


#Employee.java
 
@Entity
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    private String name;

    @Email
    private String email;

    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;

    @ManyToOne
    @JoinColumn(name = "project_id")
    private Project project;
}



#Department.java

@Entity
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    private String name;

    @OneToMany(mappedBy = "department")
    private List<Employee> employees;
}


#Project.java

@Entity
public class Project {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    private String name;

    @OneToMany(mappedBy = "project")
    private List<Employee> employees;
}


#Repositories


@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    List<Employee> findByDepartmentId(Long departmentId);
}


#Service Layer


@Service
public class EmployeeService {
    @Autowired private EmployeeRepository employeeRepository;

    @Cacheable("employees")
    public List<Employee> getAllEmployees() {
        return employeeRepository.findAll();
    }

    @CacheEvict(value = "employees", allEntries = true)
    public Employee createEmployee(Employee employee) {
        return employeeRepository.save(employee);
    }
}

#REST Controllers

 
@RestController
@RequestMapping("/employees")
public class EmployeeController {
    @Autowired private EmployeeService employeeService;

    @GetMapping
    public List<Employee> getAllEmployees() {
        return employeeService.getAllEmployees();
    }

    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Employee> createEmployee(@RequestBody Employee employee) {
        return ResponseEntity.status(HttpStatus.CREATED).body(employeeService.createEmployee(employee));
    }
}


#Security Configuration

 
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.oauth2Login()
            .and()
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.GET, "/employees/**").hasAnyRole("ADMIN", "MANAGER", "EMPLOYEE")
                .requestMatchers(HttpMethod.POST, "/employees").hasRole("ADMIN")
                .anyRequest().authenticated()
            );
        return http.build();
    }
}


#EmployeeServiceTest.java)

 
@SpringBootTest
public class EmployeeServiceTest {
    @Autowired private EmployeeService employeeService;
    @MockBean private EmployeeRepository employeeRepository;

    @Test
    void testGetAllEmployees() {
        List<Employee> employees = List.of(new Employee(1L, "John Doe", "john@example.com"));
        Mockito.when(employeeRepository.findAll()).thenReturn(employees);

        List<Employee> result = employeeService.getAllEmployees();
        assertEquals(1, result.size());
    }
}
