# Configuração do Hyper-V Replica com HTTPS (Certificados Self-Signed)

Este guia descreve, passo a passo, como configurar o **Hyper-V Replica utilizando HTTPS**, com certificados autoassinados (self-signed), em ambientes **Hyper-V Standalone ou Workgroup**.

> **Pré-requisitos**
> - Dois virtualizadores Hyper-V (Origem e Destino)
> - Comunicação de rede entre eles
> - Nome FQDN corretamente configurado em cada servidor
> - Acesso administrativo (Administrator)

---

## 1. Criar os certificados no virtualizador de origem

Todo o processo de criação de certificados inicia no **virtualizador de origem**.

---

## 2. Criar o certificado ROOT (Autoridade Certificadora interna)

Abra o **PowerShell como Administrador** e execute o comando abaixo:

```powershell
New-SelfSignedCertificate \
-Type "Custom" \
-KeyExportPolicy "Exportable" \
-Subject "HV-BROKER" \
-CertStoreLocation "Cert:\LocalMachine\My" \
-KeySpec "Signature" \
-KeyUsage "CertSign" \
-NotAfter (Get-Date).AddDays(10000)
```

Esse certificado será utilizado como **Root CA** para assinar os certificados dos virtualizadores.

---

## 3. Copiar o Thumbprint do certificado ROOT

Após a criação:
1. Abra **MMC** → *Certificates (Local Computer)*
2. Navegue até **Personal → Certificates**
3. Localize o certificado **HV-BROKER**
4. Abra o certificado e copie o **Thumbprint**

> ⚠️ Guarde esse Thumbprint, ele será utilizado no próximo passo.

---

## 4. Criar certificados para cada virtualizador (Origem e Destino)

Execute o comando abaixo **uma vez para cada servidor**, ajustando o **CN para o FQDN correto**:

```powershell
New-SelfSignedCertificate \
-type "Custom" \
-KeyExportPolicy "Exportable" \
-Subject "CN=HITDRVSGROUP.cloud.vsgroup" \
-CertStoreLocation "Cert:\LocalMachine\My" \
-KeySpec "KeyExchange" \
-TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.1,1.3.6.1.5.5.7.3.2") \
-Signer "Cert:LocalMachine\My\49C0E6AC41B6003770F6E5DDEE0220BD7D4F6A32" \
-Provider "Microsoft Enhanced RSA and AES Cryptographic Provider" \
-NotAfter (Get-Date).AddDays(10000)
```

📌 **Importante:**
- O valor do **CN** deve ser **exatamente igual ao FQDN do servidor**
- Substitua o Thumbprint pelo valor do seu certificado ROOT

---

## 5. Exportar certificados

### 5.1 Exportar certificados dos servidores (PFX)

Para **cada virtualizador**:
1. Abra **MMC → Certificates (Local Computer)**
2. Vá até **Personal → Certificates**
3. Selecione o certificado do servidor
4. Exporte no formato **PFX**
5. Defina uma **senha**

### 5.2 Exportar o certificado ROOT (CER)

1. Selecione o certificado **HV-BROKER**
2. Exporte no formato **CER**
3. No **virtualizador de destino**, importe este certificado em:
   - **Trusted Root Certification Authorities**

---

## 6. Importar os certificados PFX em ambos os virtualizadores

Copie os arquivos **PFX** para ambos os servidores e execute:

```powershell
$mypwd = Get-Credential -UserName 'Enter password below' -Message 'Enter password below'

Import-PfxCertificate \
-CertStoreLocation Cert:\LocalMachine\My\ \
-FilePath .\PIOXII-HV01.pfx \
-Password $mypwd.Password
```

🔁 Repita o processo para todos os certificados necessários.

---

## 7. Desativar verificação de revogação de certificados

Execute **em ambos os virtualizadores**:

```cmd
REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Virtualization\Replication" \
/v DisableCertRevocationCheck \
/d 1 \
/t REG_DWORD \
/f
```

---

## 8. Ajustar regras de firewall

Habilite a regra de firewall do Hyper-V Replica HTTPS **em ambos os servidores**:

```cmd
netsh advfirewall firewall set rule group="Hyper-V Replica HTTPS" new enable=yes
```

---

## 9. Habilitar o Hyper-V Replica via HTTPS

Em **cada virtualizador**:
1. Abra **Hyper-V Manager**
2. Clique em **Hyper-V Settings**
3. Vá em **Replication Configuration**
4. Marque **Enable this computer as a Replica server**
5. Selecione **Use certificate-based authentication (HTTPS)**
6. Escolha o certificado correspondente ao servidor
7. Mantenha as demais opções padrão e salve

---

## 10. Habilitar a replicação da VM

No **virtualizador de origem**:
1. Clique com o botão direito na VM
2. Selecione **Enable Replication**
3. Informe o virtualizador de destino
4. Conclua o assistente

---

## 11. Observação importante sobre portas

Caso ocorra erro durante a replicação:
- A **porta 443** pode já estar em uso
- Volte ao **Hyper-V Settings → Replication Configuration**
- Altere a porta para outra (exemplo: **8443**)

📌 Essa alteração deve ser feita **apenas no virtualizador de origem**.

---

## Referência (KB)

Artigo com mais detalhes técnicos:
- Setup 2 Hyper-V 2016 Servers – Enable Hyper-V Replica with Self-Created Certificates
https://medium.com/@pbengert/setup-2-hyper-v-2016-servers-enable-hyper-v-replica-with-self-created-certificates-and-connect-to-fceef21c8b8e

