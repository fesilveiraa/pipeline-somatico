# pipeline-somatico
Pipeline somático - Do VCF (anotado) até o CGI Classificação

****1. Clonar o git Imabrasil-hg38****

```bash
!git clone https://github.com/renatopuga/lmabrasil-hg38.git
```
output:
```Cloning into 'lmabrasil-hg38'...
remote: Enumerating objects: 226, done.
remote: Counting objects: 100% (168/168), done.
remote: Compressing objects: 100% (108/108), done.
remote: Total 226 (delta 90), reused 114 (delta 56), pack-reused 58 (from 1)
Receiving objects: 100% (226/226), 8.63 MiB | 26.63 MiB/s, done.
Resolving deltas: 100% (106/106), done.
```

Agora, vá até o github Imabrasil-hg38 na sessão Usando CGI via API Rest no Google Collab

```bash
# cortar pelas colunas de 1 a 4
#1: CHROM (converte CHROM para CHR) formato que o CGI gosta
#2: POS
#3: REF
#4: ALT
cut -f1-4 /content/lmabrasil-hg38/vep_output/liftOver_WP048_hg19ToHg38.vep.filter.tsv | sed -e "s/CHROM/CHR/g"  > df_WP048-cgi.txt
```

# listar as 10 primeiras linhas 
head df_WP048-cgi.txt
```

**Enviar job para Cancer Genome Interpreter (CGI) API**
> fonte: https://www.cancergenomeinterpreter.org/rest_api

Após filtrar apenas as colunas de interesse (CHR, POS, REF e ALT), agora podemos enviar via REST-API as variantes somaticas da amostra WP048.

>Nota: altere a variavel {SEU_TOKEN} para o token do CGI criado para sua conta.
```python
import requests
headers = {'Authorization': 'fesilveira23@gmail.com {SEU_TOKEN}'}
payload = {'cancer_type': 'HEMATO', 'title': 'Somatic MF WP048', 'reference': 'hg38'}
r = requests.post('https://www.cancergenomeinterpreter.org/api/v1',
                headers=headers,
                files={
                        'mutations': open('/content/df_WP048-cgi.txt', 'rb')
                        },
                data=payload)
r.json()
```
output:
```
be1c1bc834503fb4e668
```

**Status do job ID (Running, Error, Done)**

Verifique o status do job ID para identificar se a análise terminou ou se houve um erro.

```python
import requests
job_id ="be1c1bc834503fb4e668"

headers = {'Authorization': 'fesilveira23@gmail.com {SEU_TOKEN}'}
r = requests.get('https://www.cancergenomeinterpreter.org/api/v1/%s' % job_id, headers=headers)
r.json()
```

**Log completo do job ID**

Aqui podemos verificar o status em cada uma das etapas da análise do CGI.

```python
import requests
job_id ="be1c1bc834503fb4e668"

headers = {'Authorization': 'fesilveira23@gmail.com {SEU_TOKEN}'}
payload={'action':'download'}
r = requests.get('https://www.cancergenomeinterpreter.org/api/v1/%s' % job_id, headers=headers, params=payload)
with open('/content/results/WP048/WP048-cgi.zip', 'wb') as fd:
    fd.write(r._content)
```

**Download dos Resultados** 

Total de 4 arquivos de resultados:

> Definir cada um deles com base na documentação do CGI (todos)
- alterations.tsv: ...
- biomarkers.tsv: ...
