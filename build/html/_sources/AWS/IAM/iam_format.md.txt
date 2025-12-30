# IAMのフォーマット
公式ドキュメントの[IAMのCloudFormationのテンプレートリファレンス](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/AWS_IAM.html)


## Policy
基本的には、PolicyDocumentに作成したPolicyを記述する。
```yaml
Policy:
    Type: "AWS::IAM::ManagedPolicy"
    Properties:
        Path: "/"
        ManagedPolicyName: !Sub "policy-[xxx]"
        Description: "[Description Your Policy]"
        PolicyDocument:
            Version: "2012-10-17"
            Statement:
                - 
                    Effect: Allow
                    Action:
                        - "[Target Action]"
                    Resource: 
                        - "[Target Resource]" 
```



## Role
AssumeRolePolicyDocumentに信頼ポリシーの記述を、ManagedPolicyArnsにRoleを付与するPolicyの参照を記述する。
Policiesというセクションでインラインポリシーを作成することもできる。

```yaml
Role:
    Type: AWS::IAM::Role
    Properties:
        Path: "/"
        RoleName: !Sub "role-xxx"
        Description: "[Description Your Role]"
        MaxSessionDuration: 3600
        AssumeRolePolicyDocument:
            Version: 2012-10-17
            Statement:
                - 
                    Effect: Allow
                    Principal:
                        Service:
                            - [principal].amazonaws.com
                    Action:
                        - sts:AssumeRole
        ManagedPolicyArns: 
            - [Policy For This Role]
        Tags:
            - 
                Key: CreatedBy
                Value: !Ref "AWS::StackName"
```

