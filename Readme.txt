using AxCS.Insights.API.Services;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.OpenApi.Models;

var builder = WebApplication.CreateBuilder(args);

// ── Services ──────────────────────────────────────────────────────────────────

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

// Swagger with Bearer token support
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "AxCS Insights API",
        Version = "v1"
    });

    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name          = "Authorization",
        Type          = SecuritySchemeType.ApiKey,
        Scheme        = "Bearer",
        BearerFormat  = "JWT",
        In            = ParameterLocation.Header,
        Description   = "Enter your Bearer token"
    });

    c.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id   = "Bearer"
                }
            },
            new string[] {}
        }
    });
});

// Shared index (RAG pipeline, embeddings, chunks)
builder.Services.AddSingleton<SharedIndexService>();

// ── Authentication ────────────────────────────────────────────────────────────

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.SecurityTokenValidators.Clear();
        options.SecurityTokenValidators.Add(
            new LegacyOAuthSecurityTokenHandler(
                new LegacyTokenAuthenticationOptions
                {
                    DecryptionKey  = builder.Configuration
                                        .GetValue<string>(
                                            "LegacyTokenAuthentication:DecryptionKey"),
                    ValidationKey  = builder.Configuration
                                        .GetValue<string>(
                                            "LegacyTokenAuthentication:ValidationKey"),
                    EncryptionMethod  = EncryptionMethod.AES,
                    ValidationMethod  = ValidationMethod.HMACSHA256
                }));
    });

// ── Authorization ─────────────────────────────────────────────────────────────

builder.Services.AddAuthorization(options =>
{
    // Anyone authenticated can query
    options.AddPolicy("ApiUser", policy =>
        policy.RequireAuthenticatedUser());

    // Admin-only operations (reindex, concept management)
    options.AddPolicy("AdminOnly", policy =>
        policy.Requirements.Add(new AdminUserRequirement()));
});

builder.Services.AddScoped<IAuthorizationHandler, AdminUserHandler>();

// ── CORS (optional — needed if web UI calls this API) ─────────────────────────

builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", builder =>
    {
        builder
            .AllowAnyOrigin()
            .AllowAnyHeader()
            .AllowAnyMethod();
    });
});

// ── Build ─────────────────────────────────────────────────────────────────────

var app = builder.Build();

// ── Pipeline ──────────────────────────────────────────────────────────────────

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseCors("AllowAll");

// ORDER MATTERS: Authentication → Authorization → Controllers
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

// Load index once at startup
// (no source rescan — that's what --reindex or POST /api/admin/reindex are for)
await app.Services
         .GetRequiredService<SharedIndexService>()
         .InitializeAsync();

app.Run();

