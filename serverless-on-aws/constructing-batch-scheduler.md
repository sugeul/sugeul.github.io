## constructing batch scheduler on AWS which 

Before leaving AiRS, NAVER, I used to use Airflow to schedule batch jobs.
Airflow provides many good features: visual maintenance tool, SLA or task fail callback, scale out feature like celery executor, and useful environment variables like `ds`, `yesterday_ds`, or otherelse let job know when the job is scheduled for.
But Airflow has drawbacks also: infinitly increasing logs, become slower slower as time goes, and mainly it needs to maintain a server instance in using time.
In Vlogr, we use less reserved instance and use more SaaS and IaaS, 
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

