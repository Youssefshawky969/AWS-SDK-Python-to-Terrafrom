# AWS-Terraform Generator Tool

[section:overview]

### Overview

This tool generates Terraform configuration files for AWS VPCs by fetching VPC data using Boto3. The generated file can be used to replicate or manage VPC resources through Terraform.
It will list AWS resources (VPC, EC2) in a specified region, and generate corresponding Terraform code files for VPCs.

[section:project structure]

### Project Structure

```
project/
├── utils/
│ ├── vpc_fetcher.py                # VPC fetching logic
│ └── terraform_generator.py        # Terraform generation logic
├── terraform/
│ └── vpc.tf                        # Generated Terraform file
├── main.py                         # Main entry point
└── README.md                       # Project documentation
```

[section:implementing steps]

### Implementing Steps

Lets's begin with the `vpc_fetcher.py`, this file contains functions that connect to AWS and fetch resource data.
```
"""
aws_fetch.py

Contains utility functions that use Boto3 to list AWS VPCs and EC2 instances.
Used by the CLI tool to generate Terraform configs.
"""

from typing import List, Dict
import boto3

def list_vpcs(region: str) -> List[Dict]:
    """
    List all VPCs in the specified AWS region.

    Args:
        region (str): AWS region (e.g., 'us-east-1').

    Returns:
        List[Dict[str, Any]]: A list of VPC objects from Boto3.
    """
    ec2 = boto3.client('ec2', region_name=region)
    response = ec2.describe_vpcs()
    return response.get("Vpcs", [])

def list_ec2_instances(region: str) -> List[Dict]:
    """
    List all EC2 instances in the specified AWS region.

    Args:
        region (str): AWS region (e.g., 'us-east-1').

    Returns:
        List[Dict[str, Any]]: A list of EC2 instance details.
    """
    ec2 = boto3.client('ec2', region_name=region)
    response = ec2.describe_instances()
    instances = []
    for reservation in response['Reservations']:
        instances.extend(reservation['Instances'])
    return instances
```

After that `terraform_generator.py` this will take the VPC data and generate `.tf` files.
```
"""
tf_generator.py

Generates Terraform configuration (.tf) files based on AWS resource data.
"""

import os
from typing import List, Dict, Any

def generate_vpc_tf(vpcs: List[Dict[str, Any]], filename: str = "vpcs.tf", region: str = "us-east-1") -> None:
    """
    Generate Terraform configuration for the provided VPCs.

    Args:
        vpcs (List[Dict[str, Any]]): A list of VPC objects returned from Boto3.
        filename (str): The file path to write the Terraform config to.
        region (str): The AWS region for the provider block in Terraform.

    Returns:
        None
    """
    terraform_content = f'''provider "aws" {{
  region = "{region}"
}}\n\n'''

    for idx, vpc in enumerate(vpcs):
        cidr_block = vpc.get("CidrBlock", "0.0.0.0/0")
        tags = vpc.get("Tags", [])
        name_tag = next((tag["Value"] for tag in tags if tag["Key"] == "Name"), f"VPC_{idx + 1}")

        terraform_content += f'''
resource "aws_vpc" "vpc_{idx + 1}" {{
  cidr_block = "{cidr_block}"
  tags = {{
    Name = "{name_tag}"
  }}
}}
'''

    # Ensure output directory exists
    os.makedirs(os.path.dirname(filename) or ".", exist_ok=True)

    with open(filename, "w", encoding="utf-8") as tf_file:
        tf_file.write(terraform_content)

    print(f"Terraform file generated at: {filename}")
```

Finally, `main.py` that is the entry point that the user will run.
```
"""
This module provides functions to fetch AWS resources such as VPCs and EC2 instances
using the Boto3 SDK. It is used in the AWS-to-Terraform CLI tool to gather data
for Terraform config generation.
"""

from utils.aws_fetch import list_vpcs, list_ec2_instances
from utils.tf_generator import generate_vpc_tf

def main():
    """
    Main entry point to the CLI tool.
    Prompts user for AWS region, lists VPCs/EC2s,
    and generates a Terraform file from fetched data.
    """
    region = input("Enter AWS region (e.g. us-east-1): ").strip()
    
    # List VPCs
    vpcs = list_vpcs(region)
    print(f"Found {len(vpcs)} VPCs.")

    # List EC2s (optional)
    ec2s = list_ec2_instances(region)
    print(f"Found {len(ec2s)} EC2 instances.")

    # Generate Terraform file
    generate_vpc_tf(vpcs)
    print("Terraform file 'vpcs.tf' has been generated.")

if __name__ == "__main__":
    main()
```


**Run the script**

```
python main.py
```

** Outputs**

- If VPCs are found, a Terraform file will be generated at `terraform/vpc.tf` like this:

<img width="1025" height="315" alt="image" src="https://github.com/user-attachments/assets/973b335a-8a69-4c05-b28b-7f2b48193a5a" />




