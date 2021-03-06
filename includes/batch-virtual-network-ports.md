---
title: インクルード ファイル
description: インクルード ファイル
services: batch
documentationcenter: ''
author: LauraBrenner
manager: evansma
editor: ''
ms.assetid: ''
ms.service: batch
ms.devlang: na
ms.topic: include
ms.tgt_pltfrm: na
ms.workload: ''
ms.date: 04/03/2020
ms.author: labrenne
ms.custom: include file
ms.openlocfilehash: dc08dcded6418208751edbffcb5d263db059ec01
ms.sourcegitcommit: 62c5557ff3b2247dafc8bb482256fef58ab41c17
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 04/03/2020
ms.locfileid: "80657472"
---
### <a name="general-requirements"></a>一般的な要件

* VNET が存在するサブスクリプションとリージョンは、プールの作成に使用する Batch アカウントと同じである必要があります。

* VNET を使用するプールには最大 4,096 ノードを含めることができます。

* プールに指定されたサブネットには、プールの対象となる VM 数 (つまり、プールの `targetDedicatedNodes` および `targetLowPriorityNodes` プロパティの合計) に対応できる十分な未割り当て IP アドレスが必要です。 サブネットの未割り当て IP アドレスが十分でない場合、プールによってコンピューティング ノードが部分的に割り当てられ、サイズ変更エラーが発生します。

* VNET を提供するカスタムの DNS サーバーが、Azure Storage エンドポイントを解決できることが必要です。 具体的には、フォーム `<account>.table.core.windows.net`、`<account>.queue.core.windows.net`、`<account>.blob.core.windows.net` の URL を解決できる必要があります。

その他の VNET 要件は、Batch プールが仮想マシンの構成にあるかクラウド サービスの構成にあるかによって異なります。 VNET に新しいプールをデプロイする場合は、仮想マシンの構成が推奨されます。

### <a name="pools-in-the-virtual-machine-configuration"></a>仮想マシンの構成におけるプール

**サポートされる VNET** - Azure Resource Manager ベースの VNET のみ

**サブネット ID** - Batch API を使用してサブネットを指定するときに、そのサブネットの "*リソース識別子*" を使用します。 サブネット識別子の形式は次のとおりです。

  ```
  /subscriptions/{subscription}/resourceGroups/{group}/providers/Microsoft.Network/virtualNetworks/{network}/subnets/{subnet}
  ```

**アクセス許可** - VNET のサブスクリプションまたはリソース グループに対するロックまたはセキュリティ ポリシーで、VNET を管理するためのユーザーのアクセス許可が制限されているかどうかを確認します。

**追加のネットワーク リソース** - VNET を含んでいるリソース グループには、Batch によって自動的に追加のネットワーク リソースが割り当てられます。

> [!IMPORTANT]
>専用ノード 50 台ごと (低優先度のノードは 20 台ごと) に、1 つのネットワーク セキュリティ グループ (NSG)、1 つのパブリック IP アドレス、1 つのロード バランサーが Batch によって割り当てられます。 これらのリソースは、サブスクリプションの[リソース クォータ](../articles/azure-resource-manager/management/azure-subscription-service-limits.md)によって制限されます。 大規模なプールでは、これらの 1 つ以上のリソースについて、クォータの引き上げの要求が必要になる場合があります。

#### <a name="network-security-groups-batch-default"></a>ネットワーク セキュリティ グループ:Batch の既定値

サブネットでは、コンピューティング ノード上のタスクをスケジュールできるよう、Batch サービスからのインバウンド通信を許可する必要があります。また、ワークロードのニーズに応じて、Azure Storage などのリソースと通信するために、アウトバウンド通信を許可する必要があります。 仮想マシンの構成におけるプールの場合は、計算ノードにアタッチされたネットワーク インターフェイス (NIC) のレベルで、Batch により NSG が追加されます。 これらの NSG は、次の追加の規則で構成されます。

* `BatchNodeManagement` サービス タグに対応する Batch サービスの IP アドレスからポート 29876 と 29877 で受信するインバウンド TCP トラフィック。
* ポート 22 (Linux ノード) またはポート 3389 (Windows ノード) のインバウンド TCP トラフィック。リモート アクセスを許可するために使用されます。 Linux 上の特定の種類のマルチインスタンス タスク (MPI など) の場合、Batch コンピューティング ノードを含むサブネット 内の IP で SSH ポート 22 のトラフィックも許可する必要があります。 これは、サブネット レベルの NSG 規則に従ってブロックされる可能性があります (下記参照)。
* 仮想ネットワークに向かう全ポートのアウトバウンド トラフィック。 これは、サブネット レベルの NSG 規則に従って修正される可能性があります (下記参照)。
* インターネットに向かう全ポートのアウトバウンド トラフィック。 これは、サブネット レベルの NSG 規則に従って修正される可能性があります (下記参照)。

> [!IMPORTANT]
> Batch によって構成された NSG のインバウンド規則またはアウトバウンド規則を変更したり追加したりする際は注意が必要です。 指定したサブネット内のコンピューティング ノードとの通信が NSG によって拒否された場合、Batch サービスによって、コンピューティング ノードの状態が**使用不可**に設定されます。 また、Batch によって作成されたリソースにはリソース ロックを適用しないでください。そうしないと、プールの削除など、ユーザーが開始した操作の結果として、リソースのクリーンアップが妨げられる可能性があります。

#### <a name="network-security-groups-specifying-subnet-level-rules"></a>ネットワーク セキュリティ グループ:サブネット レベルの規則の指定

仮想ネットワークのサブネット レベルで NSG を指定する必要はありません。これは、Batch によってその独自の NSG が構成されるためです (上記参照)。 Batch の計算ノードがデプロイされているサブネットに関連付けられている NSG がある場合、またはカスタム NSG 規則を適用して既定の適用を上書きする場合は、次のテーブルに示すように、少なくとも受信および送信のセキュリティ規則を持つ NSG を構成する必要があります。

ポート 3389 (Windows) またはポート 22 (Linux) のインバウンド トラフィックは、外部ソースからの計算ノードに対するリモート アクセスを許可する必要がある場合にのみ構成します。 特定の MPI ランタイムでマルチインスタンス タスクをサポートする必要がある場合は、Linux でポート 22 の規則を有効にすることが必要になる場合があります。 これらのポートでトラフィックを許可することは、プール計算ノードを使用できるようにするために厳密には必要ありません。

**[受信セキュリティ規則]**

| ソース IP アドレス | 発信元サービス タグ | ソース ポート | 宛先 | 宛先ポート | Protocol | アクション |
| --- | --- | --- | --- | --- | --- | --- |
| 該当なし | `BatchNodeManagement` [サービス タグ](../articles/virtual-network/security-overview.md#service-tags) (リージョン バリアントを使用している場合は、Batch アカウントと同じリージョン) | * | Any | 29876 から 29877 | TCP | Allow |
| Linux マルチインスタンス タスクのためにコンピューティング ノードまたはコンピューティング ノード サブネット (あるいは両方) にリモート アクセスするためのユーザー ソース IP (必要な場合) | 該当なし | * | Any | 3389 (Windows)、22 (Linux) | TCP | Allow |

> [!WARNING]
> Batch サービスの IP アドレスは、時間の経過と共に変化することがあります。 そのため、NSG 規則には `BatchNodeManagement` サービス タグ (またはリージョン バリアント) を使用することを強くお勧めします。 直接 Batch サービスの IP アドレスを使用して NSG 規則を設定することはお勧めしません。

**アウトバウンド セキュリティ規則**

| source | ソース ポート | 宛先 | 宛先サービス タグ | 宛先ポート | Protocol | アクション |
| --- | --- | --- | --- | --- | --- | --- |
| Any | * | [サービス タグ](../articles/virtual-network/security-overview.md#service-tags) | `Storage` (リージョン バリアントを使用している場合は、Batch アカウントと同じリージョン) | 443 | TCP | Allow |

### <a name="pools-in-the-cloud-services-configuration"></a>クラウド サービスの構成におけるプール

**サポートされる VNET** - クラシック VNET のみ

**サブネット ID** - Batch API を使用してサブネットを指定するときに、そのサブネットの "*リソース識別子*" を使用します。 サブネット識別子の形式は次のとおりです。

  ```
  /subscriptions/{subscription}/resourceGroups/{group}/providers/Microsoft.ClassicNetwork /virtualNetworks/{network}/subnets/{subnet}
  ```

**アクセス許可** - `Microsoft Azure Batch` サービス プリンシパルに、指定された VNET に対する `Classic Virtual Machine Contributor` ロールベースのアクセス制御 (RBAC) のロールが付与されている必要があります。

#### <a name="network-security-groups"></a>ネットワーク セキュリティ グループ

サブネットでは、コンピューティング ノード上のタスクをスケジュールできるよう、Batch サービスからのインバウンド通信を許可する必要があります。また、Azure Storage などのリソースと通信するために、アウトバウンド通信を許可する必要があります。

NSG を指定する必要はありません。Batch IP アドレスからプールのノードへのみのインバウンド通信が Batch によって構成されるためです。 ただし、指定したサブネットに、NSG やファイアウォールが関連付けられている場合は、次の表に示すようにインバウンドとアウトバウンドのセキュリティ規則を構成します。 指定したサブネット内のコンピューティング ノードとの通信が NSG によって拒否された場合、Batch サービスによって、コンピューティング ノードの状態が**使用不可**に設定されます。

ポート 3389 (Windows) のインバウンド トラフィックは、プールのノードに対する RDP アクセスを許可する必要がある場合に構成します。 プールのノードを使用可能な状態にするうえで必ずしも必要な作業ではありません。

**[受信セキュリティ規則]**

| ソース IP アドレス | ソース ポート | 宛先 | 宛先ポート | Protocol | アクション |
| --- | --- | --- | --- | --- | --- |
Any <br /><br />実際上は "すべて許可" が必要ですが、Batch サービス以外の IP アドレスをすべてフィルターで除外する ACL 規則が、Batch サービスにより各ノードのレベルで適用されます。 | * | Any | 10100、20100、30100 | TCP | Allow |
| コンピューティング ノードへの RDP アクセスを許可する場合 (省略可能) | * | Any | 3389 | TCP | Allow |

**アウトバウンド セキュリティ規則**

| source | ソース ポート | 宛先 | 宛先ポート | Protocol | アクション |
| --- | --- | --- | --- | --- | --- |
| Any | * | Any | 443  | Any | Allow |
