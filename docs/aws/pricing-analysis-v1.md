# AWS Associated costs

Creating and configuring an Amazon SageMaker domain itself is 100% free. AWS does not charge you a base fee for keeping a domain active, nor do they charge for creating user profiles. [1] 
However, "idle" or hidden infrastructure costs can start accumulating immediately after creation, and you will be billed for any resources your users spin up inside the domain. [2] 
## 1. Underlying Networking Costs (The Main "Idle" Cost)
When you set up a domain, it requires network routing. If you choose the Quick Setup or configure it to run inside a private VPC, AWS may deploy resources on your behalf that cost money even if no one is logged in: [3, 4, 5] 

* NAT Gateways: If your domain needs to securely access the public internet, a [NAT Gateway](https://aws.amazon.com/vpc/pricing/) costs roughly $32–$36 per month per gateway (plus data processing fees), running 24/7.
* VPC Endpoints (AWS PrivateLink): Connecting your domain securely to other AWS services (like S3 or SageMaker APIs) requires endpoints. Each endpoint costs about $7–$11 per month. [4, 6] 
* Total baseline infrastructure cost can range from $100 to $350/month if these are left running continuously. [6] 

## 2. User Storage Costs (Persistent)
The moment you create a user profile and they open SageMaker Studio, AWS provisions an Amazon EFS (Elastic File System) home directory for that user. [2] 

* You are billed for the amount of data stored in these volumes (typically around $0.08 to $0.30 per GB/month, depending on the region and storage tier).
* This storage cost persists even if the user logs out or shuts down their notebooks. [2] 

## 3. Active Workload Costs (Pay-as-you-go)
Your largest and most variable expenses come from the actual machine learning workloads launched from the domain: [2, 7] 

* Notebooks & IDEs: Launching JupyterLab, Code Editor, or RStudio spins up an EC2 instance behind the scenes. For example, a basic ml.t3.medium instance costs about $0.05/hour, while heavy GPU training instances (like ml.p3.2xlarge) cost upwards of $3.06/hour. [2, 7, 8] 
* Idle Apps: If a developer forgets to shut down their JupyterLab session before going home for the weekend, the compute billing continues by the second. [2] 

## Summary of How to Prevent Surprise Bills

   1. Use Existing VPC Infrastructure: During domain setup, attach it to your existing VPC subnets and NAT Gateways instead of letting the wizard provision brand-new ones.
   2. Install an Auto-Shutdown Extension: Implement the [SageMaker Studio auto-shutdown tool](https://docs.aws.amazon.com/whitepapers/latest/sagemaker-studio-admin-best-practices/cost-attribution.html) so idle notebooks automatically turn off after a set period of inactivity (e.g., 2 hours). [9] 
   3. Monitor AWS Budgets: Tag your domain and set up [AWS Budgets alerts](https://aws.amazon.com/aws-cost-management/aws-budgets/) to ping your team if daily spend exceeds your expectations.

Would you like help choosing the right network configuration (Public Internet vs. VPC-only) to minimize these baseline infrastructure costs?

[1] [https://repost.aws](https://repost.aws/questions/QUYadK-PqMTfSD4OMSvR4ZBw/sagemaker-charging-query)
[2] [https://www.cloudexmachina.io](https://www.cloudexmachina.io/blog/sagemaker-pricing)
[3] [https://docs.aws.amazon.com](https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/create-domain-sagemaker-unified-studio-quick.html)
[4] [https://builder.aws.com](https://builder.aws.com/content/3CdN6NRi4qP4sjhHPWC0pLG6HXu/cost-breakdown-for-sagemaker-unified-studio-domain-and-project)
[5] [https://www.finout.io](https://www.finout.io/blog/amazon-sagemaker-basics-pricing-and-cost-optimization-tips)
[6] [https://builder.aws.com](https://builder.aws.com/content/3CdN6NRi4qP4sjhHPWC0pLG6HXu/cost-breakdown-for-sagemaker-unified-studio-domain-and-project)
[7] [https://www.cloudexmachina.io](https://www.cloudexmachina.io/blog/sagemaker-pricing)
[8] [https://repost.aws](https://repost.aws/questions/QUiLvCEkbITXKD-GjAGvP_yg/does-sagemaker-unified-studio-incur-additional-charges-for-using-datazone-in-the-backend)
[9] [https://docs.aws.amazon.com](https://docs.aws.amazon.com/whitepapers/latest/sagemaker-studio-admin-best-practices/cost-attribution.html)
