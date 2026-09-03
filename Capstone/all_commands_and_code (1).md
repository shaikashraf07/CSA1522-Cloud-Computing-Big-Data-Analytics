# Hadoop Log Analysis Project — All Commands & Code (Reference Only)

## 1. Fix dpkg lock error (if it occurs before any apt install)
```bash
sudo killall unattended-upgrade
sudo killall apt.systemd.daily
ps aux | grep -i apt
sudo rm /var/lib/dpkg/lock-frontend
sudo rm /var/lib/dpkg/lock
sudo dpkg --configure -a
```

## 2. Install Java
```bash
sudo apt update
sudo apt install openjdk-8-jdk -y
java -version
readlink -f $(which java)
```

## 3. Passwordless SSH to localhost
```bash
sudo apt install openssh-server -y
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 600 ~/.ssh/authorized_keys
ssh localhost
exit
```

## 4. Download and extract Hadoop
```bash
cd ~
wget https://downloads.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
tar -xzvf hadoop-3.3.6.tar.gz
sudo mv hadoop-3.3.6 /usr/local/hadoop
```

## 5. Environment variables — add to ~/.bashrc
```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export HADOOP_HOME=/usr/local/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
```
```bash
source ~/.bashrc
echo $HADOOP_HOME
```

## 6. Set JAVA_HOME inside Hadoop itself
Edit `$HADOOP_HOME/etc/hadoop/hadoop-env.sh`, set:
```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```

## 7. Hadoop config files — `$HADOOP_HOME/etc/hadoop/`

**core-site.xml** (inside `<configuration>...</configuration>`)
```xml
<property>
  <name>fs.defaultFS</name>
  <value>hdfs://localhost:9000</value>
</property>
```

**hdfs-site.xml**
```xml
<property>
  <name>dfs.replication</name>
  <value>1</value>
</property>
<property>
  <name>dfs.namenode.name.dir</name>
  <value>/usr/local/hadoop/hdfs/namenode</value>
</property>
<property>
  <name>dfs.datanode.data.dir</name>
  <value>/usr/local/hadoop/hdfs/datanode</value>
</property>
```

**mapred-site.xml**
```xml
<property>
  <name>mapreduce.framework.name</name>
  <value>yarn</value>
</property>
<property>
  <name>mapreduce.application.classpath</name>
  <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
</property>
```

**yarn-site.xml**
```xml
<property>
  <name>yarn.nodemanager.aux-services</name>
  <value>mapreduce_shuffle</value>
</property>
```

## 8. Create storage dirs, format HDFS, start services
```bash
sudo mkdir -p /usr/local/hadoop/hdfs/namenode
sudo mkdir -p /usr/local/hadoop/hdfs/datanode
sudo chown -R $USER:$USER /usr/local/hadoop
hdfs namenode -format
start-dfs.sh
start-yarn.sh
jps
```

## 9. Verify Hadoop works
```bash
hdfs dfs -mkdir /test
echo "hello hadoop" > sample.txt
hdfs dfs -put sample.txt /test
hdfs dfs -ls /test
hdfs dfs -cat /test/sample.txt
```
Web UIs: `http://<VM-IP>:9870` (HDFS), `http://<VM-IP>:8088` (YARN)

---

## 10. Module 1 — Generate sample log data (synthetic, optional)
```bash
mkdir ~/logs && cd ~/logs
python3 -c "
import random, datetime
ips = ['192.168.1.'+str(i) for i in range(1,50)]
paths = ['/home','/login','/api/data','/checkout','/product/1','/product/2']
codes = [200,200,200,301,404,500]
with open('access.log','w') as f:
    for i in range(5000):
        ip = random.choice(ips)
        t = (datetime.datetime.now() - datetime.timedelta(minutes=random.randint(0,10000))).strftime('%d/%b/%Y:%H:%M:%S +0000')
        path = random.choice(paths)
        code = random.choice(codes)
        size = random.randint(200,5000)
        f.write(f'{ip} - - [{t}] \"GET {path} HTTP/1.1\" {code} {size}\n')
"
```

## 11. Module 1 — Ingestion script
`~/scripts/ingest.sh`
```bash
#!/bin/bash
LOCAL_DIR=~/logs
HDFS_DIR=/logs/raw

hdfs dfs -mkdir -p $HDFS_DIR
for f in $LOCAL_DIR/*.log; do
  filename=$(basename "$f")
  hdfs dfs -put -f "$f" "$HDFS_DIR/$filename"
  echo "Uploaded $filename to HDFS at $(date)"
done
```
```bash
chmod +x ~/scripts/ingest.sh
~/scripts/ingest.sh
hdfs dfs -ls /logs/raw
```

Optional cron automation:
```bash
crontab -e
# add: */10 * * * * /home/yourusername/scripts/ingest.sh >> /home/yourusername/ingest.log 2>&1
```

---

## 12. Module 2 — mapper.py
```python
#!/usr/bin/env python3
import sys, re

log_pattern = re.compile(
    r'(?P<ip>\S+) \S+ \S+ \[(?P<time>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) \S+" (?P<status>\d+) (?P<size>\S+)'
)

for line in sys.stdin:
    line = line.strip()
    match = log_pattern.match(line)
    if not match:
        continue
    ip = match.group('ip')
    status = match.group('status')
    path = match.group('path')
    print(f"IP\t{ip}\t1")
    print(f"STATUS\t{status}\t1")
    print(f"PATH\t{path}\t1")
```

## 13. Module 2 — reducer.py
```python
#!/usr/bin/env python3
import sys

counts = {}
for line in sys.stdin:
    line = line.strip()
    parts = line.split('\t')
    if len(parts) != 3:
        continue
    category, key, value = parts
    combined_key = (category, key)
    counts[combined_key] = counts.get(combined_key, 0) + int(value)

for (category, key), total in sorted(counts.items()):
    print(f"{category}\t{key}\t{total}")
```

## 14. Module 2 — test locally, then run on Hadoop
```bash
cd ~/logs
cat *.log | python3 mapper.py | sort | python3 reducer.py > local_test_output.txt
head local_test_output.txt

chmod +x mapper.py reducer.py

hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-*.jar \
  -input /logs/raw \
  -output /logs/analysis_output \
  -mapper mapper.py \
  -reducer reducer.py \
  -file mapper.py \
  -file reducer.py

hdfs dfs -cat /logs/analysis_output/part-00000 | head -30
```

---

## 15. Module 3 — pull results out of HDFS
```bash
mkdir ~/dashboard && cd ~/dashboard
hdfs dfs -getmerge /logs/analysis_output/ results.tsv
```

## 16. Module 3 — load_results.py
```python
import sqlite3

conn = sqlite3.connect('log_analysis.db')
cur = conn.cursor()
cur.execute('DROP TABLE IF EXISTS results')
cur.execute('CREATE TABLE results (category TEXT, key TEXT, count INTEGER)')

with open('results.tsv') as f:
    for line in f:
        parts = line.strip().split('\t')
        if len(parts) != 3:
            continue
        category, key, count = parts
        cur.execute('INSERT INTO results VALUES (?, ?, ?)', (category, key, int(count)))

conn.commit()
conn.close()
print("Loaded into log_analysis.db")
```
```bash
python3 load_results.py
```

## 17. Module 3 — dashboard.py
```python
import streamlit as st
import sqlite3
import pandas as pd
import plotly.express as px

st.set_page_config(page_title="Log Analysis Dashboard", layout="wide")
st.title("Hadoop-Based Distributed Log Analysis Dashboard")

conn = sqlite3.connect('log_analysis.db')
df = pd.read_sql('SELECT * FROM results', conn)

tab1, tab2, tab3 = st.tabs(["Status Codes", "Top IPs", "Top Paths"])

with tab1:
    status_df = df[df['category'] == 'STATUS'].sort_values('count', ascending=False)
    st.plotly_chart(px.bar(status_df, x='key', y='count', title="Requests by HTTP Status Code"), use_container_width=True)

with tab2:
    ip_df = df[df['category'] == 'IP'].sort_values('count', ascending=False).head(15)
    st.plotly_chart(px.bar(ip_df, x='key', y='count', title="Top 15 IPs by Request Count"), use_container_width=True)

with tab3:
    path_df = df[df['category'] == 'PATH'].sort_values('count', ascending=False)
    st.plotly_chart(px.bar(path_df, x='key', y='count', title="Requests by Path"), use_container_width=True)

st.subheader("Raw Aggregated Data")
st.dataframe(df)
```
```bash
pip3 install streamlit pandas plotly
streamlit run dashboard.py --server.address 0.0.0.0
```
Open `http://<VM-IP>:8501` from host browser.

---

## 18. Full end-to-end run (after everything above is set up once)
```bash
~/scripts/ingest.sh
cd ~/logs
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-*.jar \
  -input /logs/raw -output /logs/analysis_output_v2 \
  -mapper mapper.py -reducer reducer.py \
  -file mapper.py -file reducer.py
cd ~/dashboard
hdfs dfs -getmerge /logs/analysis_output_v2/ results.tsv
python3 load_results.py
streamlit run dashboard.py --server.address 0.0.0.0
```
(Bump output folder name each run, or `hdfs dfs -rm -r /logs/analysis_output` first)

---

## 19. Reference repo (Katy-katy Web log analysis) — optional
```bash
cd ~
git clone https://github.com/Katy-katy/Web-log-analysis-using-MapReduce-Python.git
cd Web-log-analysis-using-MapReduce-Python
chmod +x my_mapper_by_page.py my_mapper_by_ip.py my_mapper_by_page2.py my_reducer.py my_reducer2.py

cat test_access_log | python3 my_mapper_by_page.py | sort | python3 my_reducer.py

hdfs dfs -mkdir -p /weblogs
hdfs dfs -put test_access_log /weblogs/

hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-*.jar \
  -input /weblogs/test_access_log -output /weblogs/output_by_page \
  -mapper my_mapper_by_page.py -reducer my_reducer.py \
  -file my_mapper_by_page.py -file my_reducer.py

hdfs dfs -cat /weblogs/output_by_page/part-00000
```

---

## 20. Checking outputs at each stage
```bash
# Module 1
hdfs dfs -ls /logs/raw

# Module 2
hdfs dfs -cat /logs/analysis_output/part-00000 | head -30

# Module 3
sqlite3 log_analysis.db
# then inside sqlite3: SELECT * FROM results LIMIT 10;
```
