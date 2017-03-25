<h3>hello, TeamPeople.</h3>

Aqui tem um exemplo e explicação de como usar autenticação por no asp.net core, um pouco diferente da versão 4.6 do asp.net, mas nada muito complicado.

Primeiro, a versão do asp.net core utilizada é a 1.1.1, foi criado um projeto padrão com login usando Identity -> Project -> ASP.NET Core Web Application (.NET Core), Authentication: Individual User Accounts. Utlizado Visual Studio 2017 RTM/Final

Os pacotes necessários são:

Install-Package System.IdentityModel.Tokens.Jwt <br>
Install-Package Microsoft.AspNetCore.Authentication.JwtBearer

Depois de instalado os pacotes, vamos modificar o arquivo appsettings.json e adicionar uma nova section, assim:

```
"Tokens": {
  "Key": "SuaSuperFodasticaChaveHere",
  "Issuer": "http://meusitehere.com",
  "Audience": "http://meusitehere.com"
}
```


Agora vamos modificar a classe Startup.cs, adicione a seguinte linha de código no método ConfigureService:

```C#
services.AddSingleton(Configuration);
```

E o seguinte bloco de código no método Configure, logo após app.UseIdentity para validar o token:

```C#
app.UseJwtBearerAuthentication(new JwtBearerOptions()
  {
    AutomaticAuthenticate = true,
    AutomaticChallenge = true,
    TokenValidationParameters = new TokenValidationParameters()
    {
      ValidIssuer = Configuration["Tokens:Issuer"],
      ValidAudience = Configuration["Tokens:Audience"],					
      ValidateIssuerSigningKey = true,
      IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(Configuration["Tokens:Key"])),
      ValidateLifetime = true
    }
  });
```

Agora no controller AccountController, adicione o seguinte ao construtor da clase:

```C#
public AccountController(...,	IConfigurationRoot config)
{
  ...
  _config = config;
            
}
```

e adicione:

```C#
private IConfigurationRoot _config;
```

crie dois novos métodos para cadastro do usuario e gerar o token:

```C#
[HttpPost]
[AllowAnonymous]
public async Task<IActionResult> RegisterWithJson([FromBody]RegisterViewModel model)
{
  try
  {
    if (!ModelState.IsValid)
    {
      return BadRequest(ModelState);
    }

    var user = new ApplicationUser { UserName = model.Email, Email = model.Email };
    var result = await _userManager.CreateAsync(user, model.Password);
    if (!result.Succeeded)
    {
      AddErrors(result);
      return BadRequest(ModelState);
    }
    return Ok();
  }
  catch (Exception ex)
  {
    return StatusCode(500, ex);
  }
}

[HttpPost]
[AllowAnonymous]
public async Task<IActionResult> Token([FromBody]LoginViewModel model)
{
  try
  {
    if (!ModelState.IsValid)
    {
      return BadRequest(ModelState);
    }

    var user = await _userManager.FindByNameAsync(model.Email);

    if (user == null)
      return NotFound();

    if (!await _userManager.CheckPasswordAsync(user, model.Password))
      return BadRequest("invalid login attempt");

    // Get user claims if you provide
    var userClams = await _userManager.GetClaimsAsync(user);

    // Get identity claims for identity
    var identityClaims = new[]
    {
      new Claim(ClaimTypes.Name, user.UserName),
      new Claim(ClaimTypes.NameIdentifier, user.Id),
      new Claim("AspNet.Identity.SecurityStamp", await _userManager.GetSecurityStampAsync(user)),
    };

    //var claims = new[]
    //{
      //Your custom claims
      //new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
    //}.Union(userClams).Union(identityClaims);

    var claims = userClams.Union(identityClaims);

    var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["Tokens:Key"]));

    var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

    var token = new JwtSecurityToken(
      issuer: _config["Tokens:Issuer"],
      audience: _config["Tokens:Audience"],
      claims: claims,
      expires: DateTime.UtcNow.AddHours(1),
      signingCredentials: creds
      );

    return Ok(new {
      token = new JwtSecurityTokenHandler().WriteToken(token),
      expiration = token.ValidTo
    });

  }
  catch (Exception ex)
  {
    return StatusCode(500, ex);
  }
}
```

Agora já está criando o token e autenticando, vamos testar a autenticação, crie um método assim no AccountController:

```
[HttpGet]
public async Task<IActionResult> GetUserName()
{
  try
  { 
    var user = await _userManager.GetUserAsync(HttpContext.User);
    return Ok(user.UserName);
  }
  catch (Exception ex)
  {
    return StatusCode(500, ex);
  }
}
```

Agora user a Collection do PostMan no link abaixo para testar:

https://github.com/jonatanrinckus/HowToUseTokenOnAspNetCore/blob/master/HowToUseTokenOnAspNetCore/HowToUseTokenOnAspNetCore.postman_collection.json

É isso, simples. :D
