> [!Info]
> Before Pulse migration, creating a system user in Airhart use pem key from a generated certificate, which the identity provider use identity4 support, but after migration, Pulse idp expect the user certificate to be in JWK format instead (openiddict)

Save this script as <name>.ps1

``` powershell
# Convert PEM or PFX certificate to JWK (JSON Web Key) format
# This script helps convert certificates for use with IdentityServer4 PrivateKeyJwtSecretValidator

param(
    [Parameter(Mandatory=$true)]
    [string]$CertificatePath,
    
    [Parameter(Mandatory=$false)]
    [string]$Password = "",
    
    [Parameter(Mandatory=$false)]
    [string]$OutputPath = "jwk-output.json"
)

# Function to convert byte array to Base64Url encoding
function ConvertTo-Base64Url {
    param([byte[]]$Bytes)
    $base64 = [Convert]::ToBase64String($Bytes)
    $base64Url = $base64.Replace('+', '-').Replace('/', '_').TrimEnd('=')
    return $base64Url
}

try {
    Write-Host "Loading certificate from: $CertificatePath" -ForegroundColor Cyan
    
    # Load the certificate
    $cert = $null
    if ($CertificatePath.EndsWith(".pfx") -or $CertificatePath.EndsWith(".p12")) {
        if ($Password) {
            $securePassword = ConvertTo-SecureString -String $Password -AsPlainText -Force
            $cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2($CertificatePath, $securePassword, [System.Security.Cryptography.X509Certificates.X509KeyStorageFlags]::Exportable)
        } else {
            $cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2($CertificatePath, "", [System.Security.Cryptography.X509Certificates.X509KeyStorageFlags]::Exportable)
        }
    } elseif ($CertificatePath.EndsWith(".pem") -or $CertificatePath.EndsWith(".crt") -or $CertificatePath.EndsWith(".cer")) {
        $certContent = Get-Content -Path $CertificatePath -Raw
        $certBytes = [System.Text.Encoding]::UTF8.GetBytes($certContent)
        $cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2($certBytes)
    } else {
        throw "Unsupported certificate format. Please use .pfx, .p12, .pem, .crt, or .cer files."
    }

    Write-Host "Certificate loaded successfully" -ForegroundColor Green
    Write-Host "Subject: $($cert.Subject)" -ForegroundColor Gray
    Write-Host "Thumbprint: $($cert.Thumbprint)" -ForegroundColor Gray
    
    # Get the public key
    $publicKey = $cert.PublicKey.Key
    
    if ($publicKey -is [System.Security.Cryptography.RSA]) {
        Write-Host "`nExporting RSA public key parameters..." -ForegroundColor Cyan
        
        $rsaParams = $publicKey.ExportParameters($false)
        
        # Create JWK
        $jwk = @{
            kty = "RSA"
            use = "sig"
            kid = $cert.Thumbprint
            n = ConvertTo-Base64Url -Bytes $rsaParams.Modulus
            e = ConvertTo-Base64Url -Bytes $rsaParams.Exponent
        }
        
        # Add x5c (certificate chain) - often required
        $x5c = [Convert]::ToBase64String($cert.RawData)
        $jwk.x5c = @($x5c)
        $jwk.x5t = ConvertTo-Base64Url -Bytes ([System.Text.Encoding]::UTF8.GetBytes($cert.Thumbprint))
        
        # Convert to JSON using System.Text.Json for proper formatting
        # Manual JSON creation to avoid PowerShell's extra spaces
        $jwkJson = @"
{
  "kty": "RSA",
  "use": "sig",
  "kid": "$($jwk.kid)",
  "n": "$($jwk.n)",
  "e": "$($jwk.e)",
  "x5c": [
    "$($jwk.x5c[0])"
  ],
  "x5t": "$($jwk.x5t)"
}
"@
        
        # Save to file
        $jwkJson | Out-File -FilePath $OutputPath -Encoding UTF8 -NoNewline
        
        Write-Host "`nJWK generated successfully!" -ForegroundColor Green
        Write-Host "Output saved to: $OutputPath" -ForegroundColor Green
        Write-Host "`n--- JWK Content ---" -ForegroundColor Yellow
        Write-Host $jwkJson
        Write-Host "`n--- Instructions ---" -ForegroundColor Cyan
        Write-Host "1. Copy the JWK content above"
        Write-Host "2. Update the certificate for the created system user with this JWK"
        
    } else {
        throw "Certificate does not contain an RSA public key. Only RSA keys are currently supported."
    }
    
} catch {
    Write-Host "`nError: $_" -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Red
    exit 1
} 
```

Run it with
```
.\name_.ps1 -CertificatePath "path\to\your-file.p12" -Password ""
```

- Get the jwk content, and use it in the certificate field when create system user
- Put the cert.p12 in the adapter, update path and clientid name with the system user client name 
```JSON
"Authentication": {  
   "GrantType": "client_credentials",  
   "Scope": "coreapi",  
   "ClientId": "hehe",  
   "ClientAssertionType": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",  
   "SigningLocalCertificatePath": "..\\haha.p12",  
   "IdentityProviderConfigurationEndpoint": "https://identityprovider.dev.pulse.netcompany.com/.well-known/openid-configuration"  
}
```