## constructing batch scheduler on AWS which 

Before leaving AiRS, NAVER, I used to use Airflow to schedule batch jobs.
Airflow provides many good features: visual maintenance tool, SLA or task fail callback, scale out feature like celery executor.<br/>
Moreover, not like Cron, it also has useful environment variables like `ds`, `yesterday_ds`, or otherelse let job know when the job is scheduled for. If I have no such environment variable then pipeline must be spaghetti if I re-run a lapsed task. I call it **Idempotent running environment**.<br/>
But Airflow has drawbacks also: infinitly increasing logs, become slower slower as time goes, and mainly it needs to maintain a server instance in using time.<br/>
In Vlogr, we use less reserved instance and use more SaaS and IaaS, <br/>
Thus I decide to find a serverless scheduler solution. **It must have at least..**

  * Scheduling function for every N minutes(/hours/days/ ...)
  * Idempotent running environment
  * visual tools(maybe, it's not important)

## candidates

currently I am using Airflow on EC2, and that means I should care the instance in short cycle(less than a week).
I'd prefer not to use such solution.
**We have some alternatives:**

  * AWS Batch
  * AWS CloudWatch
  * AWS SageMaker

 function | AWS Batch | AWS CloudWatch | AWS SageMaker
 --- |  --- |  --- |  ---
 scheduling function | yes | yes | yes
 Idempotent<br/>running environment | no | ? | no
 visual tools | no | no | no
 additional features |
 
