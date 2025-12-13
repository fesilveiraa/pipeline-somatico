# pipeline-somatico
Pipeline somático - Do VCF (anotado) até o CGI Classificação

****1. Clonar o git Imabrasil-hg38****

```bash
%%bash
!git clone https://github.com/renatopuga/lmabrasil-hg38.git
```
output:
```python
Cloning into 'lmabrasil-hg38'...
remote: Enumerating objects: 226, done.
remote: Counting objects: 100% (168/168), done.
remote: Compressing objects: 100% (108/108), done.
remote: Total 226 (delta 90), reused 114 (delta 56), pack-reused 58 (from 1)
Receiving objects: 100% (226/226), 8.63 MiB | 26.63 MiB/s, done.
Resolving deltas: 100% (106/106), done.
```

****2. Agora, vá até o github Imabrasil-hg38 na sessão Usando CGI via API Rest no Google Collab****

```bash
%%bash
# cortar pelas colunas de 1 a 4
#1: CHROM (converte CHROM para CHR) formato que o CGI gosta
#2: POS
#3: REF
#4: ALT
cut -f1-4 /content/lmabrasil-hg38/vep_output/liftOver_WP048_hg19ToHg38.vep.filter.tsv | sed -e "s/CHROM/CHR/g"  > df_WP048-cgi.txt

# Listar as 10 primeiras linhas
head df_WP048-cgi.txt
```

output: 
```
CHR	POS	REF	ALT
chr1	114716123	C	T
chr9	5073770	G	T
```

**3. Enviar job para Cancer Genome Interpreter (CGI) API**
> fonte: https://www.cancergenomeinterpreter.org/rest_api

Após filtrar apenas as colunas de interesse (CHR, POS, REF e ALT), agora podemos enviar via REST-API as variantes somáticas da amostra WP048.

>Nota: altere a variável {SEU_TOKEN} para o token do CGI criado para sua conta.
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

**4. Status do JobID (Running, Error, Done)**

Verifique o status do job ID para identificar se a análise terminou ou se houve um erro.

```python
import requests
job_id ="be1c1bc834503fb4e668"

headers = {'Authorization': 'fesilveira23@gmail.com {SEU_TOKEN}'}
r = requests.get('https://www.cancergenomeinterpreter.org/api/v1/%s' % job_id, headers=headers)
r.json()
```
output:
```
{'status': 'Done',
 'metadata': {'id': 'be1c1bc834503fb4e668',
  'user': 'fesilveira23@gmail.com',
  'title': 'Somatic MF WP048',
  'cancertype': 'HEMATO',
  'reference': 'hg38',
  'dataset': 'input.tsv',
  'date': '2025-12-06 14:12:41'}}
```


**5. Log completo do JobID**

Aqui podemos verificar o status em cada uma das etapas da análise do CGI.

```python
import requests
job_id ="be1c1bc834503fb4e668"

headers = {'Authorization': 'fesilveira23@gmail.com {SEU_TOKEN}'}
payload={'action':'logs'}
r = requests.get('https://www.cancergenomeinterpreter.org/api/v1/%s' % job_id, headers=headers, params=payload)
r.json()
```
output:
```
{'status': 'Done',
 'logs': ['# cgi analyze input.tsv -c HEMATO -g hg38',
  '2025-12-06 15:12:44,237 INFO     Parsing input01.tsv\n',
  '2025-12-06 15:12:47,767 INFO     Running VEP\n',
  '2025-12-06 15:12:48,748 INFO     Check cancer genes and consensus roles\n',
  '2025-12-06 15:12:48,825 INFO     Annotate BoostDM mutations\n',
  '2025-12-06 15:12:48,866 INFO     Annotate OncodriveMUT mutations\n',
  '2025-12-06 15:12:50,985 INFO     Annotate validated oncogenic mutations\n',
  '2025-12-06 15:12:51,123 INFO     Check oncogenic classification\n',
  '2025-12-06 15:12:51,181 INFO     Matching biomarkers\n',
  '2025-12-06 15:12:51,266 INFO     Prescription finished\n',
  '2025-12-06 15:12:51,277 INFO     Aggregate metrics\n',
  '2025-12-06 15:12:54,337 INFO     Compress output files\n',
  '2025-12-06 15:12:54,377 INFO     Analysis done\n']}
```


**6. Download dos Resultados .zip** 

Total de 4 arquivos de resultados:

> Definir cada um deles com base na documentação do CGI (todos)
1. alterations.tsv: São todas as alterações genomicas encontradas nas amostras analisadas. 
2. biomarkers.tsv: São biomarcadores relevantes que ocorreram a partir de alterações genomicas.
3. input01.tsv: Arquivos que mostram as entradas fornecidas pelo usuário. 
4. summary.txt: É um arquivo com um resumo geral da análises dos dados que obtivemos através do programa.

```bash
%%bash
#Criar diretório com o ID da amostra dentro do results
mkdir -p results/WP048
```

```python
import requests
job_id ="be1c1bc834503fb4e668"

headers = {'Authorization': 'fesilveira23@gmail.com {SEU_TOKEN}'}
payload={'action':'download'}
r = requests.get('https://www.cancergenomeinterpreter.org/api/v1/%s' % job_id, headers=headers, params=payload)
with open('/content/results/WP048/WP048-cgi.zip', 'wb') as fd:
    fd.write(r._content)
```

```bash
%%bash
!unzip /content/results/WP048/WP048-cgi.zip -d /content/results/WP048/
```

output: 
```
Archive:  /content/results/WP048/WP048-cgi.zip
  inflating: /content/results/WP048/alterations.tsv  
  inflating: /content/results/WP048/biomarkers.tsv  
  inflating: /content/results/WP048/input01.tsv  
  inflating: /content/results/WP048/summary.txt
```

**7. Visualizar a tabela alterations.tsv**

Instalar a lib pandas

```bash
!pip install pandas
```

```python
import pandas as pd
pd.read_csv('/content/results/WP048/alterations.tsv',sep='\t',index_col=False, engine= 'python')
```
output:

|index|Input ID|CHROMOSOME|POSITION|REF|ALT|CHR|POS|ALT\_TYPE|STRAND|CGI-Sample ID|CGI-Gene|CGI-Protein Change|CGI-Oncogenic Summary|CGI-Oncogenic Prediction|CGI-External oncogenic annotation|CGI-Mutation|CGI-Consequence|CGI-Transcript|CGI-STRAND|CGI-Type|
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
|0|input01\_1|1|114716123|C|T|chr1|114716123|snp|+|input01|NRAS|G13D|oncogenic \(predicted and annotated\)|driver \(boostDM: non-tissue-specific model\)|cgi,oncokb,clinvar:13901|chr1:114716123 C\>T|missense\_variant|ENST00000369535|+|SNV|
|1|input01\_2|9|5073770|G|T|chr9|5073770|snp|+|input01|JAK2|V617F|oncogenic \(annotated\)|passenger \(oncodriveMUT\)|cgi,oncokb,clinvar:14662|chr9:5073770 G\>T|missense\_variant|ENST00000381652|+|SNV|
