# Pipeline-Somático
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
---
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
---
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
---
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

---
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

---
**6. Download dos Resultados *.zip*** 

Total de 4 arquivos de resultados:

> Definir cada um deles com base na documentação do CGI (todos)
1. alterations.tsv: Contém todas as alterações genomicas encontradas que são previstas de ser Drivers nas amostras analisadas. 
2. biomarkers.tsv: São biomarcadores relevantes que ocorreram a partir da resposta tumoral em relação a certos fármacos.
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
---
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

---
**8. Tabela de variantes somaticas em cancer em diversas amostras.**
> Se o pandas já tiver sido instalado no processamento de uma das amostras, não há necessidade de instalar novamente contanto que não saia da página e o servidor não desconecte
> > Sendo elas: WP017, WP019, WP058, WP068


```python
import pandas as pd
import glob

files = glob.glob("/content/results/*/alterations.tsv")

print("Arquivos encontrados:")
for f in files:
    print(" -", f)

dfs = []

for path in files:
    sample_id = path.split("/")[-2]
    print(f"Lendo {sample_id} ...")
    df = pd.read_csv(path, sep="\t")
    df.insert(0, "input", sample_id)
    dfs.append(df)

df_final = pd.concat(dfs, ignore_index=True)

print("Tabela final criada com sucesso!")
df_final
```


output: 
| index | input | Input ID  | CHROMOSOME | POSITION  | REF | ALT | CHR  | POS       | ALT_TYPE | STRAND | CGI-Sample ID | CGI-Gene | CGI-Protein Change | CGI-Oncogenic Summary     | CGI-Oncogenic Prediction | CGI-External oncogenic annotation | CGI-Mutation              | CGI-Consequence   | CGI-Transcript      | CGI-STRAND |
|-------|-------|-----------|------------|-----------|-----|-----|------|-----------|----------|--------|---------------|----------|--------------------|---------------------------|---------------------------|-----------------------------------|---------------------------|-------------------|---------------------|------------|
| 0     | WP019 | input01_1 | 1          | 16031913  | G   | A   | chr1 | 16031913  | snp      | +      | input01       | CLCNKA   | --                 | non-protein affecting    | non-protein affecting    | NaN                               | chr1:16031913 G>A         | intron_variant   | ENST00000331433     | +          |
| 1     | WP019 | input01_2 | 1          | 149073703 | C   | T   | chr1 | 149073703 | snp      | +      | input01       | NBPF9    | --                 | non-protein affecting    | non-protein affecting    | NaN                               | chr1:149073703 C>T        | intron_variant   | ENST00000615421     | +          |
| 2 | WP019 | input01_3  | 17 | 76736877  | G | T | chr17 | 76736877  | snp | + | input01 | SRSF2   | P95H | oncogenic (predicted and annotated) | driver (oncodriveMUT) | cgi,oncokb | chr17:76736877 G>T | missense_variant | ENST00000392485 | + |
| 3 | WP019 | input01_6  | 2  | 113117932 | G | T | chr2  | 113117932 | snp | + | input01 | IL1RN   | --   | non-protein affecting               | non-protein affecting | NaN        | chr2:113117932 G>T | 5_prime_UTR_variant | ENST00000259206 | + |
| 4 | WP019 | input01_7  | 4  | 2341569   | G | C | chr4  | 2341569   | snp | + | input01 | ZFYVE28 | P76R | non-oncogenic                       | passenger (oncodriveMUT) | NaN     | chr4:2341569 G>C   | missense_variant | ENST00000290974 | + |
| 5 | WP019 | input01_11 | 6  | 158130554 | A | C | chr6  | 158130554 | snp | + | input01 | SERAC1  | --   | non-protein affecting               | non-protein affecting | NaN        | chr6:158130554 A>C | intron_variant | ENST00000647468 | + |
| 6 | WP019 | input01_13 | 7  | 111783014 | G | A | chr7  | 111783014 | snp | + | input01 | DOCK4   | --   | non-protein affecting               | non-protein affecting | NaN        | chr7:111783014 G>A | intron_variant | ENST00000445943 | + |
| 7  | WP019 | input01_14 | 9  | 5073770   | G | T | chr9  | 5073770   | snp | + | input01 | JAK2  | V617F | oncogenic (annotated)               | passenger (oncodriveMUT) | cgi,oncokb,clinvar:14662 | chr9:5073770 G>T  | missense_variant | ENST00000381652 | + |
| 8  | WP048 | input01_1  | 1  | 114716123 | C | T | chr1  | 114716123 | snp | + | input01 | NRAS  | G13D  | oncogenic (predicted and annotated) | driver (boostDM: non-tissue-specific model) | cgi,oncokb,clinvar:13901 | chr1:114716123 C>T | missense_variant | ENST00000369535 | + |
| 9  | WP048 | input01_2  | 9  | 5073770   | G | T | chr9  | 5073770   | snp | + | input01 | JAK2  | V617F | oncogenic (annotated)               | passenger (oncodriveMUT) | cgi,oncokb,clinvar:14662 | chr9:5073770 G>T  | missense_variant | ENST00000381652 | + |
| 10 | WP068 | input01_1  | 1  | 2498679   | T | G | chr1  | 2498679   | snp | + | input01 | PLCH2 | --    | non-protein affecting               | non-protein affecting | NaN | chr1:2498679 T>G | intron_variant | ENST00000378486 | + |
| 11 | WP068 | input01_3  | 11 | 17388932  | A | C | chr11 | 17388932  | snp | + | input01 | KCNJ11 | --   | non-protein affecting               | non-protein affecting | NaN | chr11:17388932 A>C | upstream_gene_variant | ENST00000339994 | + |
| 12 | WP068 | input01_4  | 11 | 56290914  | C | T | chr11 | 56290914  | snp | + | input01 | OR8H1 | R50H | non-oncogenic                       | passenger (oncodriveMUT) | NaN | chr11:56290914 C>T | missense_variant | ENST00000641600 | + |
| 13 | WP068 | input01_5  | 14 | 30877438  | T | G | chr14 | 30877438  | snp | + | input01 | COCH   | --     | non-protein affecting               | non-protein affecting | NaN | chr14:30877438 T>G  | intron_variant        | ENST00000216361 | + |
| 14 | WP068 | input01_6  | 19 | 49441413  | T | C | chr19 | 49441413  | snp | + | input01 | SLC17A7 | --   | non-protein affecting               | non-protein affecting | NaN | chr19:49441413 T>C  | 5_prime_UTR_variant   | ENST00000221485 | + |
| 15 | WP068 | input01_8  | 21 | 37147717  | A | C | chr21 | 37147717  | snp | + | input01 | TTC3   | --     | non-protein affecting               | non-protein affecting | NaN | chr21:37147717 A>C  | intron_variant        | ENST00000399017 | + |
| 16 | WP068 | input01_9  | 4  | 54733155  | A | T | chr4  | 54733155  | snp | + | input01 | KIT    | D816V  | oncogenic (predicted and annotated) | driver (boostDM: non-tissue-specific model) | cgi,oncokb,clinvar:13852 | chr4:54733155 A>T | missense_variant | ENST00000288135 | + |
| 17 | WP068 | input01_10 | 4  | 105276128 | T | C | chr4  | 105276128 | snp | + | input01 | TET2   | I1894T | oncogenic (predicted and annotated) | driver (oncodriveMUT) | clinvar:2416868 | chr4:105276128 T>C | missense_variant | ENST00000513237 | + |
| 18 | WP068 | input01_11 | 4  | 155903714 | G | T | chr4  | 155903714 | snp | + | input01 | TDO2   | --     | non-protein affecting               | non-protein affecting | NaN | chr4:155903714 G>T | 5_prime_UTR_variant   | ENST00000536354 | + |
| 19 | WP068 | input01_13 | 7  | 47837003  | G | A | chr7  | 47837003  | snp | + | input01 | PKD1L1 | T1954M | non-oncogenic                       | passenger (oncodriveMUT) | NaN | chr7:47837003 G>A | missense_variant | ENST00000289672 | + |
| 20 | WP068 | input01_14 | 7  | 151045303 | C | T | chr7  | 151045303 | snp | + | input01 | ABCB8 | P721L | non-oncogenic                       | passenger (oncodriveMUT) | NaN | chr7:151045303 C>T | missense_variant | ENST00000297504 | + |
| 21 | WP058 | input01_1 | 12 | 57185563  | T | G | chr12 | 57185563  | snp   | + | input01 | LRP1 | C2166G | non-oncogenic               | passenger (oncodriveMUT) | NaN | chr12:57185563 T>G | missense_variant   | ENST00000243077 | + |
| 22 | WP058 | input01_2 | 15 | 28272094  | A | G | chr15 | 28272094  | snp   | + | input01 | HERC2 | --    | non-protein affecting      | non-protein affecting | NaN | chr15:28272094 A>G | intron_variant     | ENST00000261609 | + |
| 23 | WP058 | input01_3 | 15 | 43206304  | T | G | chr15 | 43206304  | snp   | + | input01 | AC068724.4 | -- | non-protein affecting | non-protein affecting | NaN | chr15:43206304 T>G | intron_variant     | ENST00000563128 | + |
| 24 | WP058 | input01_4 | 19 | 12943751  | GCAGAGGCTTAAGGAGGAGGAAGAAGACAAGAAACGCAAAGAGGAGGAGGAG | - | chr19 | 12943750 | indel | + | input01 | CALR | EQRLKEEEEDKKRKEEEE364-381X | oncogenic (predicted) | driver (oncodriveMUT) | NaN | chr19:12943751-12943751 GCAGAGGCTTAAGGAGGAGGAAGAAGACAAGAAACGCAAAGAGGAGGAGGAG>- | frameshift_variant | ENST00000316448 | + |
| 25 | WP058 | input01_5 | 2  | 105892889 | T | G | chr2  | 105892889 | snp   | + | input01 | NCK2 | --    | non-protein affecting      | non-protein affecting | NaN | chr2:105892889 T>G | intron_variant     | ENST00000233154 | + |
| 26 | WP058 | input01_6 | 20 | 32434638  | - | G | chr20 | 32434638  | indel | + | input01 | ASXL1 | -642-643X | oncogenic (predicted)  | driver (oncodriveMUT) | NaN | chr20:32434638-32434639 ->G | frameshift_variant   | ENST00000375687 | + |
| 27 | WP058 | input01_7 | 7  | 124892275 | T | C | chr7  | 124892275 | snp   | + | input01 | POT1  | K39E    | non-oncogenic             | passenger (oncodriveMUT) | NaN | chr7:124892275 T>C | missense_variant   | ENST00000357628 | + |
| 28 | WP017 | input01_1 | 1  | 114716127 | C | T | chr1  | 114716127 | snp   | + | input01 | NRAS  | G12S    | oncogenic (predicted and annotated) | driver (boostDM: non-tissue-specific model) | cgi,clinvar:177778 | chr1:114716127 C>T | missense_variant | ENST00000369535 | + |
| 29 | WP017 | input01_2 | 1  | 152304661 | G | C | chr1  | 152304661 | snp   | + | input01 | FLG   | R3409G  | non-oncogenic             | passenger (oncodriveMUT) | NaN | chr1:152304661 G>C | missense_variant   | ENST00000368799 | + |
| 30 | WP017 | input01_3 | 11 | 115209621 | G | A | chr11 | 115209621 | snp   | + | input01 | CADM1  | T344I  | non-oncogenic             | passenger (oncodriveMUT) | NaN | chr11:115209621 G>A | missense_variant   | ENST00000331581 | + |
| 31 | WP017 | input01_4 | 12 | 132140028 | C | T | chr12 | 132140028 | snp   | + | input01 | DDX51 | --     | non-protein affecting  | non-protein affecting | NaN | chr12:132140028 C>T | intron_variant    | ENST00000397333 | + |
| 32 | WP017 | input01_5 | 14 | 24119817  | G | A | chr14 | 24119817  | snp   | + | input01 | DCAF11 | R338H  | non-oncogenic           | passenger (oncodriveMUT) | NaN | chr14:24119817 G>A | missense_variant | ENST00000446197 | + |
| 33 | WP017 | input01_6 | 14 | 59727407  | G | A | chr14 | 59727407  | snp   | + | input01 | RTN1  | A426V  | non-oncogenic           | passenger (oncodriveMUT) | NaN | chr14:59727407 G>A | missense_variant | ENST00000267484 | + |
| 34 | WP017 | input01_7 | 15 | 24675943  | G | A | chr15 | 24675943  | snp   | + | input01 | NPAP1 | A26T   | non-oncogenic           | passenger (oncodriveMUT) | NaN | chr15:24675943 G>A | missense_variant | ENST00000329468 | + |
| 35 | WP017 | input01_8 | 16 | 67616834  | G | A | chr16 | 67616834  | snp   | + | input01 | CTCF  | E348K  | oncogenic (predicted)   | driver (oncodriveMUT)  | NaN | chr16:67616834 G>A | missense_variant | ENST00000646076 | + |
| 36 | WP017 | input01_9  | 16 | 71389851  | G | A | chr16 | 71389851  | snp   | + | input01 | CALB2 | E268K | non-oncogenic              | passenger (oncodriveMUT) | NaN | chr16:71389851 G>A | missense_variant | ENST00000302628 | + |
| 37 | WP017 | input01_10 | 16 | 74452195  | A | C | chr16 | 74452195  | snp   | + | input01 | GLG1  | --    | non-protein affecting     | non-protein affecting | NaN | chr16:74452195 A>C | intron_variant   | ENST00000205061 | + |
| 38 | WP017 | input01_11 | 19 | 12943751  | GCAGAGGCTTAAGGAGGAGGAAGAAGACAAGAAACGCAAAGAGGAGGAGGAG | - | chr19 | 12943750 | indel | + | input01 | CALR | EQRLKEEEEDKKRKEEEE364-381X | oncogenic (predicted) | driver (oncodriveMUT) | NaN | chr19:12943751-12943751 GCAGAGGCTTAAGGAGGAGGAAGAAGACAAGAAACGCAAAGAGGAGGAGGAG>- | frameshift_variant | ENST00000316448 | + |
| 39 | WP017 | input01_12 | 19 | 35673516  | A | C | chr19 | 35673516  | snp   | + | input01 | UPK1A | T147P | non-oncogenic              | passenger (oncodriveMUT) | NaN | chr19:35673516 A>C | missense_variant | ENST00000616789 | + |
| 40 | WP017 | input01_13 | 19 | 45319445  | T | G | chr19 | 45319445  | snp   | + | input01 | CKM   | --    | non-protein affecting     | non-protein affecting | NaN | chr19:45319445 T>G | intron_variant   | ENST00000221476 | + |
| 41 | WP017 | input01_14 | 2  | 137450992 | G | A | chr2  | 137450992 | snp   | + | input01 | THSD7B | R1036Q | non-oncogenic              | passenger (oncodriveMUT) | NaN | chr2:137450992 G>A | missense_variant | ENST00000272643 | + |
| 42 | WP017 | input01_15 | 2  | 218880905 | A | C | chr2  | 218880905 | snp   | + | input01 | WNT10A | --     | non-protein affecting     | non-protein affecting | NaN | chr2:218880905 A>C | 5_prime_UTR_variant | ENST00000258411 | + |
| 43 | WP017 | input01_16 | 20 | 32434638  | - | G | chr20 | 32434638  | indel | + | input01 | ASXL1  | -642-643X | oncogenic (predicted) | driver (oncodriveMUT) | NaN | chr20:32434638-32434639 ->G | frameshift_variant | ENST00000375687 | + |
| 44 | WP017 | input01_17 | 21 | 41901750  | C | T | chr21 | 41901750  | snp   | + | input01 | C2CD2  | --     | non-oncogenic              | passenger (oncodriveMUT) | NaN | chr21:41901750 C>T | splice_acceptor_variant | ENST00000380486 | + |
| 45 | WP017 | input01_18 | 21 | 43094667  | T | G | chr21 | 43094667  | snp   | + | input01 | U2AF1  | Q157P  | oncogenic (predicted and annotated) | driver (boostDM: non-tissue-specific model) | cgi,oncokb,clinvar:376024 | chr21:43094667 T>G | missense_variant | ENST00000291552 | + |
| 46 | WP017 | input01_20 | 3  | 133380804 | G | A | chr3  | 133380804 | snp   | + | input01 | TMEM108 | D365N | non-oncogenic              | passenger (oncodriveMUT) | NaN | chr3:133380804 G>A | missense_variant | ENST00000321871 | + |
| 47 | WP017 | input01_22 | 7  | 584561    | G | A | chr7  | 584561    | snp   | + | input01 | PRKAR1B | T239M | non-oncogenic              | passenger (oncodriveMUT) | NaN | chr7:584561 G>A | missense_variant | ENST00000406797 | + |
