---
title: Deploy a Test Cluster
summary: Use CockroachDB's CloudFormation template to deploy a Kubernetes-orchestrated test cluster on AWS.
toc: false
---

This page shows you the easiest way to test an insecure, multi-node CockroachDB cluster, using CockroachDB's [AWS CloudFormation](https://aws.amazon.com/cloudformation/) template to simplify setup and [Kubernetes](https://kubernetes.io/) to automate deployment, scaling, maintenance, and load balancing of client workloads.

{{site.data.alerts.callout_success}}To evaluate pre-release functionality from <a href="https://github.com/cockroachdb/cockroach/wiki/Roadmap">our roadmap</a>, use the <a href="../v2.0/deploy-a-test-cluster.html">v2.0 version</a> of this page.{{site.data.alerts.end}}

<div id="toc"></div>

## Before You Begin

Before getting started, it's important to review some limitations and requirements.

### Limitations

{{site.data.alerts.callout_danger}}The <a href="https://github.com/cockroachdb/cockroachdb-cloudformation">CockroachDB AWS CloudFormation template</a> is designed for testing, not for production use.{{site.data.alerts.end}}

- You can scale the cluster to a maximum of 15 nodes.

- While the AWS region for your deployment is configurable, the cluster runs in a single AWS availability zone within that region. It will easily survive and recover from node failures as long as you deploy at least 3 nodes, but it will not survive an availability zone outage.
    - For production resiliency, the recommendation would be to span 3 or more availability zones in a single region or 3 or more regions.

- The cluster is completely insecure, which comes with risks:
    - There is no network encryption or authentication, and thus no confidentiality.
    - The cluster is open to any client by default, although you have the option to restrict client access to a specific range of IP addresses.
    - Any user, even `root`, can log in without providing a password.
    - Any user, connecting as `root`, can read or write any data in your cluster.

### Requirements

- You must have an [AWS account](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-sign-up-for-aws.html).
- You must have [SSH access](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html) in the AWS region where the cluster is deployed.   

## Step 1. Start CockroachDB

1. [Launch the CockroachDB CloudFormation template](http://amzn.to/2CZjJLZ).

2. In the CloudFormation UI, review and customize the settings for the cluster. Most of the defaults are sufficient for testing scenarios. However, it's important to select your **SSH Key** so you'll be able to connect to the Kubernetes master node later, and to confirm that the **CockroachDB Version** is **v1.1.4**.

    You may also want to:
    - Change the **AWS region** where the cluster will run. The default region is **US West**. Note that some instance types may not be available in some regions.
    - Add an **IP Address Whitelist** to restrict user access to the CockroachDB admin UI and SQL client access to the cluster. By default, all locations have access.
    - Increase the initial **Cluster Size**. The default is **3** nodes.

3. When you're ready to start the cluster, click **Create**.

    The launch process generally takes 10 to 15 minutes. Once you see the `CREATE_COMPLETE` status in the CloudFormation UI, the cluster is ready for testing.

    {{site.data.alerts.callout_info}}If the launch process times out or fails, you could be running into an <a href="https://docs.aws.amazon.com/general/latest/gr/aws_service_limits.html">AWS service limit</a>. You can view any errors in the <a href=" https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-monitor-stack.html" data-proofer-ignore>event history</a>.{{site.data.alerts.end}}

## Step 2. Test the cluster

1. [Install CockroachDB](install-cockroachdb.html) on your local machine, if you haven't already.

2. In the **Outputs** section of the CloudFormation UI, note the **Connection String**.

3. In a terminal, start the [SQL shell](use-the-built-in-sql-client.html) built into the `cockroach` binary, using the **Connection String** as the `--url` flag:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql \
    --insecure \
    --url="postgresql://root@Cockroach-ApiLoadB-LVZZ3VVHMIDA-1266691548.us-west-2.elb.amazonaws.com:26257?application_name=cockroach&sslmode=disable"
    ~~~

    ~~~
    # Welcome to the cockroach SQL interface.
    # All statements must be terminated by a semicolon.
    # To exit: CTRL + D.
    #
    # Server version: CockroachDB CCL v1.1.4 (linux amd64, built 2018/01/08 17:32:42, go1.8.3) (same version as client)
    # Cluster ID: bc181e48-da0b-4336-a7a2-a041e195ab5c
    #
    # Enter \? for a brief introduction.
    #
    root@Cockroach-ApiLoadB-LVZZ3VVHMIDA-1266691548.us-west-2.elb.amazonaws.com:26257/>
    ~~~

4. Run some basic [CockroachDB SQL statements](learn-cockroachdb-sql.html):

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE bank;
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > INSERT INTO bank.accounts VALUES (1, 1000.50);
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > SELECT * FROM bank.accounts;
    ~~~

    ~~~
    +----+---------+
    | id | balance |
    +----+---------+
    |  1 |  1000.5 |
    +----+---------+
    (1 row)
    ~~~

{{site.data.alerts.callout_success}}With the <code>cockroach</code> binary on your local machine, other client <a href="cockroach-commands.html"><code>cockroach</code> commands</a> can be run in the same way.{{site.data.alerts.end}}

## Step 3. Start a load generator

CockroachDB provides a number of [example load generators](https://github.com/cockroachdb/loadgen) in Go for simulating client workloads. The program you'll use here is called [`rand`](https://github.com/cockroachdb/loadgen/tree/master/rand). It will generate and insert random data into the `bank.accounts` table you created in the previous step.

1. In a new terminal, install [Go](https://golang.org/dl/). If you're on a Mac and using Homebrew, use `brew install go`.

2. Install the `rand` load generator:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ go get github.com/cockroachdb/loadgen/rand
    ~~~

3. Start the `rand` load generator:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ rand --host=Cockroach-ApiLoadB-LVZZ3VVHMIDA-1266691548.us-west-2.elb.amazonaws.com bank accounts
    ~~~
    - Set the `--host` flag to just the hostname portion of the **ConnnectionString** from the CloudFormation UI.
    - Be sure to list `bank accounts` **after** the `--host` flag.

4. Back in the terminal with the SQL shell, check the random data that `rand` is inserting into the `bank.accounts` table:

    ~~~ sql
    > SELECT * FROM bank.accounts LIMIT 10;
    ~~~

    ~~~
    +-------+---------+
    |  id   | balance |
    +-------+---------+
    |     1 | 1000.50 |
    |  5250 |      84 |
    |  6205 |      60 |
    | 13086 |       4 |
    | 16745 |      12 |
    | 19124 |      85 |
    | 25573 |      54 |
    | 26957 |      46 |
    | 31117 |      65 |
    | 37195 |      59 |
    +-------+---------+
    (10 rows)
    ~~~

5. Exit the SQL shell:

    {% include copy-clipboard.html %}
    ~~~ sql
    > \q
    ~~~

## Step 4. Monitor the cluster

You can use the cluster's [Admin UI](admin-ui-overview.html) to monitor the `rand` workload and overall cluster behavior.

1. In the **Outputs** section of the CloudFormation UI, click the **Web UI** link.

2. On the **Cluster Overview** page, hover over the **SQL Queries** graph to see the proportion of reads and writers coming from the `rand` program.

    <img src="{{ 'images/cloudformation_admin_ui_sql_queries.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

3. Scroll down and hover over the **Replicas per Node** graph to see how CockroachDB automatically replicates your data behind-the-scenes.

    <img src="{{ 'images/cloudformation_admin_ui_replicas.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

4. Explore other areas of the [Admin UI](admin-ui-overview.html).

5. Learn more about [production monitoring and alerting](monitoring-and-alerting.html).

## Step 5. Simulate node failure

Kubernetes ensures that the cluster always has the number of nodes you specified during initial configuration (3 by default). When a node fails, Kubernetes automatically creates another node with the same network identity and persistent storage.

To see this in action:

1. In the **Outputs** section of the CloudFormation UI, note the **SSHProxyCommand**.

2. In a new terminal, run the **SSHProxyCommand** to SSH into the Kubernetes master node. Be sure to update the `SSH_KEY` environment variable definition to point to the location of your `.pem` file.

3. List the Kubernetes pods that map to CockroachDB nodes:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods
    ~~~

    ~~~
    NAME            READY     STATUS    RESTARTS   AGE
    cockroachdb-0   1/1       Running   0          1h
    cockroachdb-1   1/1       Running   0          1h
    cockroachdb-2   1/1       Running   0          1h
    ~~~

4. Kill one of CockroachDB nodes:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl delete pod cockroachdb-2
    ~~~

    ~~~
    pod "cockroachdb-2" deleted
    ~~~

5. In the Admin UI, the **Summary** panel may show one node as **Suspect**. As Kubernetes auto-restarts the node, watch how the node once again becomes healthy.

    You can also select the **Runtime** dashboard and see the restarting of the node in the **Live Node Count** graph.

    <img src="{{ 'images/cloudformation_admin_ui_live_node_count.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 6. Scale the cluster

To scale the cluster, you need to first adjust the AWS Auto Scaling group size and then the Kubernetes replica count.

1. In the **Resources** section of the CloudFormation UI, find **K8sStack** and click the corresponding **Physical ID** link.

2. In the **Resources** section of the new screen, find **K8sNodeGroup** and click the corresponding **Physical ID** link.

3. In the final screen, click **Edit**, update the **Desired** field to the new number of nodes you want in the cluster, and click **Save**.

4. In the terminal where you SSHed into the Kubernetes master node, use `kubectl` to scale your cluster to match your Auto Scaling group size:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl scale statefulsets cockroachdb --replicas=5
    ~~~

    ~~~
    statefulset "cockroachdb" scaled
    ~~~

5. In the Admin UI, check the **Replicas per Node** graph again to see how CockroachDB automatically rebalances your data evenly across all nodes.

    <img src="{{ 'images/cloudformation_admin_ui_rebalancing.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 7. Stop the cluster

Once you're done with your test cluster:

1. Stop the `rand` load generator running locally, if you haven't already.

2. In the CloudFormation UI, select **Other Actions** > **Delete Stack**. This last step is essential for deleting all AWS resources tied to your cluster. If you don't delete these resources, AWS will continue to charge you for them.

## See Also

{% include prod_deployment/prod-see-also.md %}
