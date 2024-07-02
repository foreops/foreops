---
title: "Understanding and Implementing DORA Metrics"
description: "DORA metrics and Jenkins"
lead: "DORA (DevOps Research and Assessment) metrics serve as a crucial checklist for tech teams, helping them gauge their efficiency in software development and deployment. These metrics are derived from extensive research aimed at understanding what drives successful tech teams."
summary: "DORA (DevOps Research and Assessment) metrics serve as a crucial checklist for tech teams, helping them gauge their efficiency in software development and deployment. These metrics are derived from extensive research aimed at understanding what drives successful tech teams."
date: 2024-06-30T21:13:59-04:00
lastmod: 2024-06-30T21:13:59-04:00
draft: false
weight: 50
categories: []
tags: ["dora","metrics","devops","jenkins","dashboard","flask"]
contributors: ["Sharjeel Aziz"]
pinned: false
homepage: true
images: ["dora-dashboard-og.png"]
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---

The Four Key Metrics:

1. **Deployment Frequency** - Tracks how often software is successfully released to production.
2. **Lead Time for Changes** - Measures the time taken from a commit to successful deployment in production.
3. **Mean Time to Restore (MTTR)** - Assesses how quickly a team can recover from a failure in the production environment.
4. **Change Failure Rate** - Indicates the percentage of changes to production that result in degraded service or subsequently require remediation.

These metrics come from a big study that looked at what makes tech teams successful. Although the number of metrics is low they encompass the concept of shipping quality software often and increase reliability. The metrics help teams figure out where they can get better at building and fixing their software quickly and without too many issues. You can do a quick assessment by [taking the  DORA quick check](https://dora.dev/quickcheck/). Most of the data and metrics required for implementing DORA metrics are readily available in commonly used DevOps tools and implementing them can as easy following a few steps.

Implementing DORA Metrics:

**Steps to Follow**

1. **Identify Data Sources:** Utilize version control systems, CI/CD pipelines, and monitoring tools to gather necessary data.
2. **Establish Data Collection:** Set up automated scripts or use APIs and webhooks for seamless data integration.
3. **Define Measurement Criteria:** Clearly define what constitutes each metric to ensure consistent and accurate measurement.
4. **Calculate Metrics:** Use collected data to compute metrics regularly, aggregating data over meaningful intervals like daily, weekly, or monthly.
5. **Visualize and Report:** Employ tools like Grafana or Kibana to create dashboards that present these metrics in an accessible format to stakeholders.
6. **Analyze and Act:** Regularly review metrics to identify improvement areas, using insights to drive strategic decisions and optimize processes.
7. **Continuous Improvement:** Treat the metric implementation as an ongoing process, refining methods and tools as needed to enhance accuracy and relevance.

Jenkins is commonly used for CI/CD by many development teams. Let's see if we can gather DORA metrics from a Jenkins job, we can leverage the Jenkins API and extract the relevant data. If you would like to create a test Jenkins instance rather than trying this in your production instance, please refer to this [repository](https://github.com/sharjeelaziz/blog/tree/8146a8970d3dff57f86069b8f2b750ba58e6d153/dora/jenkins-test-instance) to create a test Jenkins instance. The repository has a script for a Hello World job that fails randomly to generate data, etc.  

### Creating a Simple Dashboard

Let's put together a simple dashboard using Flask. This dashboard will connect to Jenkins through its API to gather build data, compute the Change Failure Rate, and then display these results on a web page. For detailed code, please check this [repository](https://github.com/sharjeelaziz/blog/tree/050e7c585e7a0f505866e68afa33e32be582210b/dora/metrics-dashboard).

Jenkins offers a remote access API for many of its objects, found at `/.../api/`, where "..." represents the specific object you want to access. For example, adding `/api/json` to the URL of our job lets us pull its builds in JSON format. Initially, this won't include the results of the builds, but you can get more detailed data, including build results, by using the depth=N query parameter. To see build results, you might use a URL like `http://192.168.85.235:8080/job/Hello%20World/api/json?depth=1`.

You may need to look at permissions if the API is not accessible.

Configure Security Settings:

* Go to Manage Jenkins > Security.
* Under Authorization, make sure Allow anonymous read access is enabled if it’s a test environment. In a production setting, ensure that the correct user permissions are set to allow API read access.

Test API Accessibility:

* You can check if the API is reachable for your job by simply adding /api/json to the job's URL. For instance, you might run a command like `curl http://192.168.85.235:8080/job/Hello%20World/api/json` or enter the URL directly in your web browser.

#### Fetching Build Data

The heart of our dashboard is the app.py script, which fetches and processes Jenkins build data. We use the Jenkins API to get detailed information about each build, including whether each build succeeded or failed. Here's how we fetch the build details:

```python
def get_builds(base_url, job_name):
    """
    Fetch builds information for a specific job using the Jenkins API with depth=1.
    """
    api_url = f"{base_url}/job/{job_name}/api/json?depth=1"
    response = requests.get(api_url)
    builds = response.json()["builds"]
    return builds
```

We call this function with parameters specifying the base URL of your Jenkins server and the job name. The depth=1 query parameter in the API call ensures we retrieve detailed information about each build.

#### Processing Build Data

Once we have the build data, we need to organize it by month and calculate the Change Failure Rate for each month:

```python
def aggregate_data_by_month(builds):
    """
    Organize build data by month and ensure all last six months are displayed.
    """
    last_six_months = generate_last_six_months()
    data = {
        month: {"total_builds": 0, "failed_builds": 0, "rate": 0.0}
        for month in last_six_months
    }

    for build in builds:
        timestamp = build.get("timestamp",0)
        if timestamp:
            date = datetime.fromtimestamp(timestamp / 1000.0, timezone.utc)
            month = date.strftime("%Y-%m")
            if month in data:
                data[month]["total_builds"] += 1
                if build["result"] == "FAILURE":
                    data[month]["failed_builds"] += 1

    # Calculate the change failure rate for each month
    for month in data:
        total = data[month]["total_builds"]
        failed = data[month]["failed_builds"]
        data[month]["rate"] = (failed / total * 100) if total > 0 else 0

    return [{"month": month, **data[month]} for month in last_six_months]
```

This function organizes the build data by month and calculates the total number of builds, the number of failed builds, and the Change Failure Rate.

#### Displaying the Data

The dashboard.html template uses Chart.js to visually represent the Change Failure Rate over the last six months:

```html
<script>
const ctx = document.getElementById('changeFailureRateChart').getContext('2d');
const changeFailureRateChart = new Chart(ctx, {
   type: 'line',
   data: {
         labels: {{ month_data | map(attribute='month') | list | tojson }},
         datasets: [{
            label: 'Change Failure Rate',
            data: {{ month_data | map(attribute='rate') | list | tojson }},
            borderColor: 'rgb(75, 192, 192)',
            backgroundColor: 'rgba(75, 192, 192, 0.5)',
            yAxisID: 'y',
         }]
   },
...
...
...
... 
</script>
```

This script creates a line chart displaying the Change Failure Rate for each month, making it easy to track trends and identify issues. We package all this in a lightweight container for our Flask application, ensuring that all dependencies are installed and the service is ready to run. The code for the the dashboard is available in this [repository](https://github.com/sharjeelaziz/blog/tree/050e7c585e7a0f505866e68afa33e32be582210b/dora/metrics-dashboard).

#### Build the Docker Image

Navigate to the `metric-dashboard` directory containing the Dockerfile and build the Docker image:

```bash
docker build -t jenkins-metrics-dashboard:latest .
```

#### Run the Dashboard

Run the Docker container to start the dashboard:

```bash
docker run --env JENKINS_BASE_URL="http://192.168.85.235:8080" --env JENKINS_JOB_NAME="Hello World" --name jenkins-metrics-dashboard -p 5000:5000 -d jenkins-metrics-dashboard:latest
```

#### Viewing the Dashboard

After starting the container, open your web browser and navigate to `http://localhost:5000` to view your Jenkins Metrics Dashboard. You should see the Change Failure Rate and other metrics displayed based on the data fetched from Jenkins.

{{< img src="dora-dashboard.png" alt="screenshot of dora dashboard page" caption="Dora Dashboard" class="wide" >}}

Remember, implementing DORA metrics requires collaboration between development, operations, and other relevant teams. It's essential to foster a culture of continuous improvement and use the metrics to drive positive changes in your software delivery processes. Here are some ideas on how to extract other metrics.

**Extract Deployment Frequency:**

* Analyze the job's build history to determine the deployment frequency.
* Iterate through the builds and count the number of successful builds within a specific time period (e.g., daily, weekly, or monthly).
* Calculate the deployment frequency based on the count and the chosen time period.

**Extract Lead Time for Changes:**

* For each successful build, retrieve the timestamp of when the code change was committed (usually available in the version control system) and the timestamp of when the build was completed.
* Calculate the lead time by subtracting the commit timestamp from the build completion timestamp.
* Aggregate the lead times for all successful builds within the desired time period to calculate the average lead time.

**Extract Mean Time to Restore (MTTR):**

* Identify the builds that resulted in failures or required a rollback.
* For each failed build, retrieve the timestamp of when the failure occurred and the timestamp of when the issue was resolved (i.e., a successful build after the failure).
* Calculate the time to restore by subtracting the failure timestamp from the resolution timestamp.
* Aggregate the time to restore values for all failures within the desired time period to calculate the MTTR.

**Extract Change Failure Rate:**

* Count the total number of builds within a specific time period.
* Count the number of builds that resulted in failures or required a rollback within the same time period.
* Calculate the change failure rate by dividing the number of failed builds by the total number of builds.

**Store and Visualize Metrics:**

* Store the extracted DORA metrics in a database or a metrics storage system.
* Use a visualization tool or create custom dashboards to present the metrics in a meaningful way.
* Update the metrics regularly by scheduling the data extraction process to run periodically (e.g., daily or weekly).

To automate the process of gathering DORA metrics from Jenkins, you can use tools like Jenkins Pipeline or create custom scripts (e.g., using Python or Groovy) that interact with the Jenkins API and perform the necessary calculations.

Additionally, there are Jenkins plugins and third-party tools that can help with DORA metrics collection and visualization.

##### Code
* [Jenkins Test Instance](https://github.com/sharjeelaziz/blog/tree/8146a8970d3dff57f86069b8f2b750ba58e6d153/dora/jenkins-test-instance)
* [Metrics Dashboard](https://github.com/sharjeelaziz/blog/tree/050e7c585e7a0f505866e68afa33e32be582210b/dora/metrics-dashboard)