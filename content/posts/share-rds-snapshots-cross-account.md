---
author: minwoo.kim
categories:
  - AWS
date: 2021-12-26T15:48:34Z
tags:
  - AWS
  - RDS
title: 'AWS 계정간 RDS 스냅샷 공유'
cover:
  image: '/assets/img/aws-smile.jpg'
  alt: 'AWS Smile'
  relative: false
---

AWS에서는 계정간 database snapshot을 공유할 수 있는 기능을 제공합니다. 해당 내용을 진행하기 위해서는 아래와 같이 진행하면 됩니다.

Database가 존재하는 계정을 **A**, 스냅샷을 공유받을 계정을 **B**라고 가정하겠습니다.

1. **A 계정**에서 Database snapshot 생성합니다.
2. (Encrypted인 경우) snapshot을 copy 하기 위해 **A 계정**에서 KMS key를 생성합니다.
3. (Encrypted인 경우) **A 계정**에서 생성된 KMS key에 **B 계정**을 공유할 수 있도록 설정합니다.

   ![[Add other AWS accounts] 를 통해 AWS account ID 를 추가합니다.](/assets/post/39e6ff31-2dd3-54dd-831c-4d77a689bc09.png)

   [Add other AWS accounts] 를 통해 AWS account ID 를 추가합니다.

4. **A 계정**에서 공유할 snapshot을 copy 합니다.

   ![e4ad8711-30a8-507b-817e-00fd589272d1.png](/assets/post/e4ad8711-30a8-507b-817e-00fd589272d1.png)

   ![Master Key 항목에서 생성한 KMS키를 선택합니다.](/assets/post/cca04507-d9b2-5882-89af-4ab6ac238cd6.png)

   Master Key 항목에서 생성한 KMS키를 선택합니다.

5. **A 계정**에서 Snapshot copy가 완료되면, Share snapshot을 통해 공유합니다.

   ![39395701-ee70-5346-87ba-a50ff5d6cb1e.png](/assets/post/39395701-ee70-5346-87ba-a50ff5d6cb1e.png)

   ![e6c6e297-84dd-59c0-b748-031d9ce82853.png](/assets/post/e6c6e297-84dd-59c0-b748-031d9ce82853.png)

   1. AWS account ID 입력 후, Add
   2. Save를 눌러 공유

6. **B 계정**의 RDS → Snapshots → **Shared with me** 에 가서 공유받은 Snapshot을 다시 Copy합니다.

   ![e4a3fdf5-168d-5135-a4a8-5ffd62795d8f.png](/assets/post/e4a3fdf5-168d-5135-a4a8-5ffd62795d8f.png)

   ![Master Key는 B 계정에서 사용할 Key로 선택합니다. (aws/rds 권장)](/assets/post/f862d76e-f9e3-5b18-86a7-cbbd773f9179.png)

   Master Key는 B 계정에서 사용할 Key로 선택합니다. (aws/rds 권장)

7. **B 계정**에서 Snapshot copy가 완료되면 RDS→ Snapshots → **Manual** 에 가서 해당 스냅샷을 Restore snapshot을 통해 RDS Migration 완료합니다.

   ![c45263ed-1ef6-5c28-9c17-e7c31068e15b.png](/assets/post/c45263ed-1ef6-5c28-9c17-e7c31068e15b.png)
