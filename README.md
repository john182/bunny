# spring-boot-bunny
Este � um componente multitenant para ser usado em aplica��es spring-boot.
Com este componente, pode ser configurado m�ltiplas fontes de dados para manter os dados em diferentes esquemas.
Que est� usando o [Hibernate suporte multi-tenancy] (https://docs.jboss.org/hibernate/orm/4.2/devguide/en-US/html/ch16.html) trabalhando com a estrat�gia de banco de dados separado.

## Configura��o
1) Baixar o projeto no git https://github.com/betao22ster/bunny, dar o install do maven.

2) No seu projeto, adicionar a dependencia:

```xml
	<dependency>
		<groupId>com.msv</groupId>
		<artifactId>bunny</artifactId>
		<version>0.0.4-SNAPSHOT</version>
	</dependency>
```
		
3) No seu projeto, criar uma entity que implementa a interface DataSourceConfig.
EX:

```java
@Entity
@Table(name="NOME_TABLE")
public class DataSourceTable implements DataSourceConfig, Serializable {

	private static final long serialVersionUID = -5018185835788890513L;
	
	@Id @GeneratedValue
    private Long id;
    private String name;
    private String url;
    private String username;
    private String password;
    private String driverClassName;
    private boolean initialize;

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public String getUrl() {
        return url;
    }

    public String getUsername() {
        return username;
    }

    public String getPassword() {
        return password;
    }

    public String getDriverClassName() {
        return driverClassName;
    }

    public boolean getInitialize() {
        return initialize;
    }
}
```

4) No seu projeto, criar uma respository que implementa a interface DataSourceConfigRepository.

```java

@Repository
public interface DataSourceTableRepository extends DataSourceConfigRepository<DataSourceTable>, JpaRepository<DataSourceTable, Long> {

}

```

## Configura��o do Spring Security
Porque integrar com o Spring Security?<br>
Quando o sistema � integrado ao Spring Security, ap�s o login, os dados do usu�rio ficam registrados.<br> 
� poss�vel obter os dados do login como est� abaixo ou injetando.<br>

```java
	SecurityContextHolder.getContext().getAuthentication().getPrincipal();
```

Desta forma, podemos ter neste objeto, os dados que o bunny precisa.<br>
Na classe TenantIdentifierResolver do bunny ele identifica se o j� existe um usu�rio logado, e j� retorna o identificador do tenancy.<br>
Para nos isso � importante. 
<br>
Caso esteja em um projeto de micro-service, os demais servi�os, iram j� funcionar corretamente, identificando o tenancy pelo usu�rio logado.<br>

Vamos configurar o spring security:
1) Deve ser usado no maven a seguinte dependencia.

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

2) Crie uma classe que vai servir para manter nosso dados de login.

```java
public class UserDetailsModify extends User implements DataSourceConfigSecurity {

	private static final long serialVersionUID = 3375988537129735478L;
	private DataSourceTable dataSourceTable;

	public UserDetailsModify(String login, String senha, List<GrantedAuthority> auth, DataSourceTable dataSourceTable) {
		super(login, senha, auth);
		this.dataSourceTable = dataSourceTable;
	}

	@Override
	public DataSourceConfig getDataSourceConfig() {
		return dataSourceTable;
	}
	
}
```

O importante nesta classe, para o bunny, � a implementa��o da classe DataSourceConfigSecurity.

3) Crie um service b�sico, para consultar os dados do login.

```java
@Service
public class Users implements UserDetailsService {

	@Autowired
	private UserRepository repo;

	@Autowired
	private DataSourceTableRepository repository;
	
	@Override
	public OpsUserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
		
		// sua entidade da tabela de usu�rio normal
		Usuario user = repo.findByEmail(email);
		
		if (user == null) {
			return null;
		}
		
		List<GrantedAuthority> auth = AuthorityUtils.commaSeparatedStringToAuthorityList("ROLE_USER");
		
		DataSourceTable dataSourceTable = repository.findOne(user.getId());
		
		return new UserDetailsModify(user.getEmail(), user.getSenha(), auth, dataSourceTable);
	}

}
```

4) Pronto, basta criar agora o bean para configurar o spring security.

```java
	@Configuration
	@Order(SecurityProperties.ACCESS_OVERRIDE_ORDER)
	protected static class SecurityConfiguration extends WebSecurityConfigurerAdapter {

		@Autowired
		private Users users;		
		
		@Autowired
		public void globalUserDetails(AuthenticationManagerBuilder auth) throws Exception {
			// @formatter:off	
			
			auth.userDetailsService(users);
			
		}
	}
```

## Executando 
Na sua classe principal, adicionar a anota��o @LoadDataSourceConfig adicionando as duas classes criadas e a anota��o @ComponentScan com o pacote do bunny.

```java
@LoadDataSourceConfig(config=DataSourceTable.class, configRepository=DataSourceTableRepository.class)
@ComponentScan(basePackages={"org.springframework.boot.bunny.multitenant"})
```


## Agradecimentos

Ao github https://github.com/rcandidosilva. 
O projeto spring-boot-multitenant foi usado como base para este componente.






