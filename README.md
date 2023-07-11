# tsdump_multi_ansible

```tsdump_multi_ansible.yaml``` is a simple Ansible playbook to get data from one or more CockroachDB clusters for offline review.

Information that is collected includes:
- time series values from ```cockroach debug tsdump```
- statement statistics from ```system.statement_statistics```
- transaction statistics from ```system.transaction_statistics```

## Features

- Works with a mixture of secure and insecure clusters
- Output is stored in a single directory, organized by host
- You can specify how far back in time to collect statistics (default 3 days)

## Example
```
$ ansible-playbook -v -i my_inventory.ini tsdump_multi_ansible.yaml
```

where ```my_inventory.ini``` lists at least 1 host in the ```cockroachdb``` group, for example:

```
[cockroachdb]

# insecure, command in $PATH, getting stats going back 3 days:
18.224.151.57 ansible_connection=ssh ansible_user=ubuntu

# secure, command in $PATH, getting stats going back 3 days:
3.133.103.205 ansible_connection=ssh ansible_user=ubuntu cert_dir=/home/ubuntu/certs

# insecure, command not in $PATH, getting stats going back 3 days:
14.207.56.189 ansible_connection=ssh ansible_user=ubuntu cockroach_dir=/usr/local/bin

# secure, command not in $PATH, getting stats going back 3 days:
4.116.109.112 ansible_connection=ssh ansible_user=ubuntu cert_dir=/home/ubuntu/certs cockroach_dir=/usr/local/bin

# insecure, command in $PATH, getting stats going back 7 days:
18.224.151.33 ansible_connection=ssh ansible_user=ubuntu from_datetime_spec="7 days ago"
```

## Details

### Inventory

The Ansible inventory should list ONE host for each CockroachDB cluster.  As described below, you 
can specify the following Ansible variables for each host:

- ```cockroach_dir```
- ```cert_dir```
- ```from_datetime_spec```

### Specifying the directory for the ```cockroach``` command

- By default, the playbook assumes that the ```cockroach``` command is in the path for the user for
the host in the Ansible inventory.
- If the ```cockroach``` command is NOT in the path for the user for the host in the Ansible inventory, then
specify the directory that contains the ```cockroach``` command via the ```cockroach_dir``` variable for that host.

### Specifying the certificate directory for secure clusters

- By default, the playbook assumes that the cluster is running in insecure mode.
- For secure clusters, specify the ```cert_dir``` variable for the host in the Ansible inventory.
This should specify the path of the certificates directory on the host.

### Specifying the date-time from which to collect stats

- By default, all data is collected from the time range that starts 3 days prior to running the playbook
and ends at the current time.
- To change the start time, specify the ```from_datetime_spec``` variable for the host in the Ansible inventory.
The format should be compatible with the ```date``` command ```--date``` option.

Note about this ```date``` command:
- The specified date can be absolute or relative.
- The date command referred to here is the Linux ```date``` command, not the MacOS ```date``` command.

## Output

There are 4 output files for each host:

1. Output of ```cockroach debug tsdump```
2. Selected content of the table ```system.statement_statistics```
3. Selected content of the table ```system.transaction_statistics```
4. A file that maps store IDs to node IDs

Each of these output files is copied from each host and is stored on the
machine that runs this Ansible script, stored in a separate directory for each host.
The name of this directory is the name of the host from the Ansible inventory.

For example, when running this playbook with two hosts in the Ansible inventory
the output file/directory structure (as shown by the ```tree``` command) would look like:

```
▶ tree ts_top_level
ts_top_level
├── 3.143.229.15
│   ├── ss.csv.gz
│   ├── ts.csv.gz
│   ├── tsdump.gob.yaml
│   └── tsdump.raw.gz
└── 3.144.216.145
    ├── ss.csv.gz
    ├── ts.csv.gz
    ├── tsdump.gob.yaml
    └── tsdump.raw.gz

2 directories, 8 files
```

### Specifying the top-level output directory

All the above output directories are stored under a single higher-level directory on the
host on which this playbook is run.

- By default, this is ./ts_top_level
- You can change this by specifying the ```ts_top_level``` variable.

## Assumptions

- This playbook assumes the ```gzip``` command is installed on each host in the Ansible inventory,
and that it is in the user's shell command search path.
