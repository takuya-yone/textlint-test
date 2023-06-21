AWSTemplateFormatVersion: "2010-09-09"
Description: Configure AWS SSO PermissionSet

Parameters: 
  SsoInsanceId: 
    Type: String
    Description: 'xxxxxxxxxxxxxxxx from arn:aws:sso:::instance/ssoins-xxxxxxxxxxxxxxxx'
  SecurityAccountId:
    Type: String
  LogAccountId:
    Type: String
  UserManagementAccountId:
    Type: String
  AllowAddress:
    Type: String
    Description: 'input like "IPAddress-1","IPAdress-2",...'
    Default: '"1.1.1.1/32","2.2.2.2/32","3.3.3.3/32"'

Resources:
# Permission Set
  nriqumoaOrganizationsControl:
    Type: AWS::SSO::PermissionSet
    Properties:
      Description: 'Permission set being used to manege AWS organizations services'
      SessionDuration: 'PT6H' # The length of time that the application user sessions are valid for in the ISO-8601 standard.
      InstanceArn: !Sub arn:aws:sso:::instance/ssoins-${SsoInsanceId}
      Name: 'nri-qumoa-OrganizationsControl'
      InlinePolicy:  !Sub '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowListDesctibe",
                "Effect": "Allow",
                "Action": [
                    "organizations:ListRoots",
                    "organizations:ListDelegatedServicesForAccount",
                    "organizations:DescribeAccount",
                    "organizations:DescribePolicy",
                    "organizations:ListChildren",
                    "organizations:ListCreateAccountStatus",
                    "organizations:DescribeOrganization",
                    "organizations:DescribeOrganizationalUnit",
                    "organizations:DescribeHandshake",
                    "organizations:DescribeCreateAccountStatus",
                    "organizations:ListPoliciesForTarget",
                    "organizations:DescribeEffectivePolicy",
                    "organizations:ListTargetsForPolicy",
                    "organizations:ListTagsForResource",
                    "organizations:ListAWSServiceAccessForOrganization",
                    "organizations:ListPolicies",
                    "organizations:ListDelegatedAdministrators",
                    "organizations:ListAccountsForParent",
                    "organizations:ListHandshakesForOrganization",
                    "organizations:ListHandshakesForAccount",
                    "organizations:ListAccounts",
                    "organizations:ListParents",
                    "organizations:ListOrganizationalUnitsForParent",
                    "organizations:CreateOrganizationalUnit",
                    "organizations:CreatePolicy",
                    "organizations:MoveAccount"
                ],
                "Resource": "*"
            },
            {
                "Sid": "AllowOUSCPWrite",
                "Effect": "Allow",
                "Action": [
                    "organizations:UpdateOrganizationalUnit",
                    "organizations:DeleteOrganizationalUnit",
                    "organizations:UpdatePolicy",
                    "organizations:AttachPolicy",
                    "organizations:DetachPolicy",
                    "organizations:DeletePolicy",
                    "organizations:TagResource",
                    "organizations:UntagResource"
                ],
                "Resource": "*",
                "Condition": {
                    "StringNotEquals": {
                        "aws:ResourceTag/nri-qumoa": "managed-by-nri-qumoa"
                    }
                }
            },
            {
                "Sid": "AllowOrganizationsIntegrationFunctions",
                "Effect": "Allow",
                "Action": [
                    "health:*",
                    "trustedadvisor:*"
                ],
                "Resource": "*"
            },
            {
                "Effect": "Deny",
                "Action": "*",
                "Resource": "*",
                "Condition": {
                    "Bool": {
                        "aws:ViaAWSService": "false"
                    },
                    "NotIpAddress": {
                        "aws:SourceIP": [
                        ${AllowAddress}
                        ]
                    }
                }
            }
        ]
    }'

  nriqumoaSSOAdmin:
    Type: AWS::SSO::PermissionSet
    Properties:
      Description: 'Permission set being used set used for Administrator Access for AWS SSO AWS services'
      SessionDuration: 'PT6H' # The length of time that the application user sessions are valid for in the ISO-8601 standard.
      InstanceArn: !Sub arn:aws:sso:::instance/ssoins-${SsoInsanceId}
      Name: 'nri-qumoa-SSOAdmin'
      ManagedPolicies:
        - 'arn:aws:iam::aws:policy/ReadOnlyAccess'
      InlinePolicy: !Sub
        - | 
          {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowSsoOperations",
                    "Effect": "Allow",
                    "Action": [
                        "account:ListRegions",
                        "identitystore:*",
                        "ds:*",
                        "sso:*",
                        "sso-directory:*",
                        "access-analyzer:ValidatePolicy"
                    ],
                    "Resource": [
                        "*"
                    ]
                },
                {
                    "Sid": "DenyManagementAccountSsoOperations",
                    "Effect": "Deny",
                    "Action": [
                        "sso:*"
                    ],
                    "Resource": [
                        "arn:aws:sso:::account/${SecurityAccountId}",
                        "arn:aws:sso:::account/${LogAccountId}",
                        "arn:aws:sso:::account/${UserManagementAccountId}"
                    ]
                },
                {
                    "Sid": "DenyControlTowerUserAdminsOperations",
                    "Effect": "Deny",
                    "Action": [
                        "sso:PutInlinePolicyToPermissionSet",
                        "sso:DeleteInlinePolicyFromPermissionSet",
                        "sso:DetachManagedPolicyFromPermissionSet",
                        "sso:AttachManagedPolicyToPermissionSet",
                        "sso:UpdatePermissionSet"
                    ],
                    "Resource": [
                        "${nriqumoaOrganizationsControlArn}",
                        "${nriqumoaCloudTrailLogViewArn}",
                        "${nriqumoaSecurityOperationArn}",
                        "${nriqumoaLogReadOnlyArn}"
                    ]
                },
                {
                    "Effect": "Deny",
                    "Action": "*",
                    "Resource": "*",
                    "Condition": {
                        "Bool": {
                            "aws:ViaAWSService": "false"
                        },
                        "NotIpAddress": {
                          "aws:SourceIP": [
                          ${AllowAddress}
                          ]
                        }
                    }
                }
            ]
          }
        - nriqumoaOrganizationsControlArn: !GetAtt nriqumoaOrganizationsControl.PermissionSetArn
          nriqumoaCloudTrailLogViewArn: !GetAtt nriqumoaCloudTrailLogView.PermissionSetArn
          nriqumoaSecurityOperationArn: !GetAtt nriqumoaSecurityOperation.PermissionSetArn
          nriqumoaLogReadOnlyArn: !GetAtt nriqumoaLogReadOnly.PermissionSetArn

  nriqumoaCloudTrailLogView:
    Type: AWS::SSO::PermissionSet
    Properties:
      Description: 'Permission set being used for ReadOnlyAccess for AWS CloudTrail services'
      SessionDuration: 'PT6H' # The length of time that the application user sessions are valid for in the ISO-8601 standard.
      InstanceArn: !Sub arn:aws:sso:::instance/ssoins-${SsoInsanceId}
      Name: 'nri-qumoa-CloudTrailLogView'
      ManagedPolicies:
        - 'arn:aws:iam::aws:policy/CloudWatchLogsReadOnlyAccess'
      InlinePolicy: !Sub '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Deny",
                "Action": "*",
                "Resource": "*",
                "Condition": {
                    "Bool": {
                        "aws:ViaAWSService": "false"
                    },
                    "NotIpAddress": {
                        "aws:SourceIP": [
                        ${AllowAddress}
                        ]
                    }
                }
            }
        ]
    }'

  nriqumoaSecurityOperation:
    Type: AWS::SSO::PermissionSet
    Properties:
      Description: 'Permission set being used for ReadOnlyAccess for Security Account'
      SessionDuration: 'PT6H' # The length of time that the application user sessions are valid for in the ISO-8601 standard.
      InstanceArn: !Sub arn:aws:sso:::instance/ssoins-${SsoInsanceId}
      Name: 'nri-qumoa-SecurityOperation'
      ManagedPolicies:
        - 'arn:aws:iam::aws:policy/ReadOnlyAccess'
      InlinePolicy: !Sub '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowSecurityOperation",
                "Effect": "Allow",
                "Action": [
                    "access-analyzer:ApplyArchiveRule",
                    "access-analyzer:CreateArchiveRule",
                    "access-analyzer:DeleteArchiveRule",
                    "access-analyzer:StartResourceScan",
                    "access-analyzer:UpdateArchiveRule",
                    "access-analyzer:UpdateFindings",
                    "config:PutOrganizationConfigRule",
                    "config:PutOrganizationConformancePack",
                    "guardduty:ArchiveFindings",
                    "guardduty:CreateFilter",
                    "guardduty:DeleteFilter",
                    "guardduty:UnarchiveFindings",
                    "guardduty:UpdateFilter",
                    "securityhub:BatchUpdateFindings",
                    "securityhub:BatchGetStandardsControlAssociations"
                ],
                "Resource": [
                    "*"
                ]
            },
            {
                "Effect": "Deny",
                "Action": "*",
                "Resource": "*",
                "Condition": {
                    "Bool": {
                        "aws:ViaAWSService": "false"
                    },
                    "NotIpAddress": {
                        "aws:SourceIP": [
                        ${AllowAddress}
                        ]
                    }
                }
            }
        ]
    }'

  nriqumoaLogReadOnly:
    Type: AWS::SSO::PermissionSet
    Properties:
      Description: 'Permission set being used for ReadOnlyAccess for logs'
      SessionDuration: 'PT6H' # The length of time that the application user sessions are valid for in the ISO-8601 standard.
      InstanceArn: !Sub arn:aws:sso:::instance/ssoins-${SsoInsanceId}
      Name: 'nri-qumoa-LogReadOnly'
      ManagedPolicies:
        - 'arn:aws:iam::aws:policy/ReadOnlyAccess'
      InlinePolicy: !Sub '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowRunQueryOperation",
                "Effect": "Allow",
                "Action": [
                    "athena:StartQueryExecution",
                    "athena:StopQueryExecution",
                    "athena:UpdateWorkGroup",
                    "athena:UpdatePreparedStatement",
                    "athena:UpdateNamedQuery",
                    "athena:DeleteNamedQuery",
                    "athena:DeletePreparedStatement",
                    "athena:CreateNamedQuery",
                    "athena:CreatePreparedStatement"
                ],
                "Resource": "*"
            },
            {
                "Sid": "AllowRunQueryOperationForS3",
                "Effect": "Allow",
                "Action": [
                    "s3:PutObject"
                ],
                "Resource": [
                    "arn:aws:s3:::nri-qumoa-log-for-athena-query-${LogAccountId}/*"
                ]
            },
            {
                "Effect": "Deny",
                "Action": "*",
                "Resource": "*",
                "Condition": {
                    "Bool": {
                        "aws:ViaAWSService": "false"
                    },
                    "NotIpAddress": {
                        "aws:SourceIP": [
                        ${AllowAddress}
                        ]
                    }
                }
            }
        ]
     }'

  nriqumoaBillingControl:
    Type: AWS::SSO::PermissionSet
    Properties:
      Description: 'Permission set being used for ReadOnlyAccess for AWS Billing services'
      SessionDuration: 'PT6H' # The length of time that the application user sessions are valid for in the ISO-8601 standard.
      InstanceArn: !Sub arn:aws:sso:::instance/ssoins-${SsoInsanceId}
      Name: 'nri-qumoa-BillingControl'
      ManagedPolicies:
        - 'arn:aws:iam::aws:policy/ReadOnlyAccess'
        - 'arn:aws:iam::aws:policy/AWSBillingReadOnlyAccess'
        - 'arn:aws:iam::aws:policy/AWSBillingConductorFullAccess'
      InlinePolicy: !Sub '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Deny",
                "Action": "*",
                "Resource": "*",
                "Condition": {
                    "Bool": {
                        "aws:ViaAWSService": "false"
                    },
                    "NotIpAddress": {
                        "aws:SourceIP": [
                            "1.1.1.1/32",
                        ]
                    }
                }
            }
        ]
    }'
