# Authorization Code Grant 

"Public clients" a mi parecer, pues he quitado "secret" cuando envía "clientid,code a /token" ... e una webapp no puedes tener almacenado el secret porque no es un confidential app, sino un public app.

## Without PKCE

```mermaid
sequenceDiagram
	Usuario->>Cliente: Intenta acceder a un recurso protegido
	Cliente-->>IDP: Envía petición a "/authorize"
	IDP-->>Usuario: IDP redirige a página de login
	Usuario->>IDP: Se loguea y obtiene acceso
	IDP-->>Cliente: Devuelve el "code"
	Cliente->>IDP: Envia clientId, code a "/token"
	IDP-->>IDP: valida code, clientId
	IDP-->>Cliente: Si todo bien: Devuelve id, access y refresh token.
	Cliente->>API: Operación a API securizada con bearer token
```

## With PKCE

"Enviar en primera instancia un dato aleatorio que cifrará el target y que tendré que proveer tras login junto con el <<code>> que me da para verifique que quien solicita los tokens es capaz de enviar tanto el primer dato aleatorio como el que ha recuperado cifrado y el code"

```mermaid
sequenceDiagram
	Usuario->>Cliente: Intenta acceder a un recurso protegido
	Cliente-->>IDP: Envía petición a "/authorize" con hash code_challenge
	IDP-->>Usuario: IDP guarda code_challenge y redirige a página de login
	Usuario->>IDP: Se loguea y obtiene acceso
	IDP-->>Cliente: Devuelve el "code"
	Cliente->>IDP: Envia clientId, code y code_verifier a "/verification"
	IDP-->>IDP: valida clientId, code y que code_verifier === code_challenge
	IDP-->>Cliente: Si todo bien: Devuelve id, access y refresh token.
	Cliente->>API: Operación a API securizada con bearer token
	
	
```

