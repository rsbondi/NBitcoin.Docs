# Introduction

What a long road. I started writing NBitcoin in 2014 in order to learn about Bitcoin, this was being done by copy and pasting code from BitcoinJ and Bitcoin Core, along with their test suite and making sure it worked in .NET.

Along the road I learned about Bitcoin. I am the kind of programmer who can't learn something without coding it first.
As I learned about Bitcoin, I started writing [Programming The Blockchain in C#](https://programmingblockchain.gitbooks.io/programmingblockchain/content/) on a word document. Along the road, nopara73, joined me and transfered it to Github and GitBook and rewrote lot's of content.

Since then lot's of community member improved the book, that's the beauty of github.

If you want to learn conceptually about Bitcoin, I advise you to read [Programming The Blockchain in C#](https://programmingblockchain.gitbooks.io/programmingblockchain/content/) first before reading this documentation.

[Programming The Blockchain in C#](https://programmingblockchain.gitbooks.io/programmingblockchain/content/) introduces you primitive concept of Bitcoin which are useful whichever language you end up writing bitcoin.

This book is about programming with NBitcoin you will learn additional concept, not covered in [Programming The Blockchain in C#](https://programmingblockchain.gitbooks.io/programmingblockchain/content/), and mainly how to use them in NBitcoin.

NBitcoin is a tool I use everyday for other projects, be it closed source project or open source project (like [BTCPay Server](https://docs.btcpayserver.org)). This mean that there is a very tight feedback loop.

The process is about:
* Add a new feature on NBitcoin
* Use such feature for another project
* Finding the pain points of using this feature
* Fixing NBitcoin to make the use case easier to code

This feedback loop sometimes take 10 min. This is why NBitcoin has today 604 releases.

This means that you can be sure that NBitcoin is the easiest library to use for bitcoin development. If it was not, I would shoot myself in the foot.

This book will allow me to explain the design of NBitcoin which has been shaped by my direct experience into using NBitcoin.

This book is about practical development, so I expect you to open visual studio, understand concepts, run and debug the code snippets.
Once you understand, you need to apply the concept on whatever project you want to work on.

# Development setup

Because this book is meant to be used for people using Windows, Linux or Mac OS, we advise you to use [Visual studio Code](https://code.visualstudio.com/).

Download and install the last version of [.NET Core SDK](https://dotnet.microsoft.com/download).

Then create your first project and add `NBitcoin` and `NBitcoin.TestFramework`.
```bash
mkdir NBitcoinTraining
cd NBitcoinTraining
dotnet new console
dotnet add package NBitcoin
dotnet add package NBitcoin.TestFramework
```

Then open the folder with Visual Studio Code.

```bash
code .
```

You can then put breakpoints and debug from inside Visual studio code. You can hit F5 to run your code.
This might ask you to install the extension `C# for Visual Studio Code (powered by OmniSharp)`.

# NBitcoin's test framework: Alice sends through Bob through Bitcoin RPC

When developing on Bitcoin, it is critical to be able to easily run your code to verify it works properly.
This is what we will use in all our tests. This is the goal of the `NBitcoin.TestFramework` package.

This library allows you to create a network of node and connect them to one another.
Let's see an example, it may takes a while to run at first because Bitcoin Core binaries, please feel free to debug step by step:

`Miner` give some money to `Alice` which gives 1 BTC to `Bob`.

```csharp
using System;
using System.Threading;
using NBitcoin;
using NBitcoin.Tests;

namespace NBitcoinTraining
{
    class Program
    {
        static void Main(string[] args)
        {
            // During the first run, this will take time to run, as it download bitcoin core binaries (more than 40MB)
            using (var env = NodeBuilder.Create(NodeDownloadData.Bitcoin.v0_18_0, Network.RegTest))
            {
                // Removing node logs from output
                env.ConfigParameters.Add("printtoconsole", "0");

                var alice = env.CreateNode();
                var bob = env.CreateNode();
                var miner = env.CreateNode();
                env.StartAll();
                Console.WriteLine("Created 3 nodes (alice, bob, miner)");

                Console.WriteLine("Connect nodes to each other");
                miner.Sync(alice, true);
                miner.Sync(bob, true);

                Console.WriteLine("Generate 101 blocks so miner can spend money");
                var minerRPC = miner.CreateRPCClient();
                miner.Generate(101);
                
                var aliceRPC = alice.CreateRPCClient();
                var bobRPC = bob.CreateRPCClient();
                var bobAddress = bobRPC.GetNewAddress();

                Console.WriteLine("Alice get money from miner");
                var aliceAddress = aliceRPC.GetNewAddress();
                minerRPC.SendToAddress(aliceAddress, Money.Coins(20m));

                Console.WriteLine("Mine a block and check that alice is now synched with the miner (same block height)");
                minerRPC.Generate(1);
                alice.Sync(miner);

                Console.WriteLine($"Alice Balance: {aliceRPC.GetBalance()}");

                Console.WriteLine("Alice send 1 BTC to bob");
                aliceRPC.SendToAddress(bobAddress, Money.Coins(1.0m));
                Console.WriteLine($"Alice mine her own transaction");
                aliceRPC.Generate(1);

                alice.Sync(bob);

                Console.WriteLine($"Alice Balance: {aliceRPC.GetBalance()}");
                Console.WriteLine($"Bob Balance: {bobRPC.GetBalance()}");
            }
        }
    }
}
```

This will output the following:

```
Created 3 nodes (alice, bob, miner)
Connect nodes to each other
Generate 101 blocks so miner can spend money
Alice gets money from miner
Mine a block and check that alice is now synched with the miner (same block height)
Alice Balance: 20.00000000
Alice send 1 BTC to bob
Alice mine her own transaction
Alice Balance: 18.99996680
Bob Balance: 1.00000000
```

We encourage you to run this example several time, try to play with the code.
One trick you will often see is that when `Alice` send money to Bob, she mines her own transaction and make sure Bob is aware of this block.

This is done this way because when Alice creates a transaction, it would take time to propagate to `Bob`, so we would need to put some sleep in the code. Instead, we just get `Alice` to mine her own transaction and make sure `Bob` received it. Then we can be sure the balance has been updated.

You will use this trick a lot during your development.

# Debugging inside NBitcoin

NBitcoin is published with symbols so that Visual Studio Code can fetch sources from github and you can easily debug inside NBitcoin library easily.

## In Visual Studio Code

Inside your `.vscode/launch.json`, add the following to .NET Core Launch (console) configuration:
```json
"justMyCode": false,
"symbolOptions": {
    "searchPaths": [ "https://symbols.nuget.org/download/symbols" ],
    "searchMicrosoftSymbolServer": true
},
```

For example, here is my own `launch.json`:

```json
{
   "version": "0.2.0",
   "configurations": [
        {
            "name": ".NET Core Launch (console)",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "build",
            "program": "${workspaceFolder}/bin/Debug/netcoreapp2.2/NBitcoinTraining.dll",
            "args": [],
            "cwd": "${workspaceFolder}",
            "console": "internalConsole",
            "stopAtEntry": false,
            "justMyCode": false,
            "symbolOptions": {
                "searchPaths": [ "https://symbols.nuget.org/download/symbols" ],
                "searchMicrosoftSymbolServer": true
            }
        },
        {
            "name": ".NET Core Attach",
            "type": "coreclr",
            "request": "attach",
            "processId": "${command:pickProcess}"
        }
    ]
}
```

Now, when I debug into Visual Studio Code, I can step inside NBitcoin's source code. (with `F11` while breaking)

We advise you to re-enable `Enable Just My Code` when you are done. Because Loading symbols can be quite time consuming.

## In Visual Studio

You need to run at least Visual Studio 15.9. Then, you need to:

* `Go in Tools / Options / Debugging / General` and turn **off** `Enable Just My Code`.
* `Go in Tools / Options / Debugging / Symbols` and add `https://symbols.nuget.org/download/symbols` to the `Symbol file (.pdb) locations`, make sure it is checked.

You should also check `Microsoft Symbol Server` or your debugging experience in visual studio will be slowed down.

Now you can Debug your project and step inside any call to NBitcoin under visual studio.

We advise you to re-enable `Enable Just My Code` when you are done. Because Loading symbols can be quite time consuming.

# Alice sends through Bob without RPC

In the previous section, we saw how `Alice` can send to `Bob` with the `NBitcoin.TestFramework` library.
This is all well and good, but in many cases, Bitcoin RPC can be very limited and inflexible in terms of features.

So let's try to achieve the same goal as in the previous chapter, but this time, without using RPC on Alice's side.

Let's change the diff and the whole source code:

```diff
diff --git a/Program.cs b/Program.cs
index 144a0a3..b9b4983 100644
--- a/Program.cs
+++ b/Program.cs
@@ -1,4 +1,6 @@
 ﻿using System;
+using System.Linq;
+using System.Collections.Generic;
 using System.Threading;
 using NBitcoin;
 using NBitcoin.Tests;
@@ -15,42 +17,68 @@ namespace NBitcoinTraining
                 // Removing node logs from output
                 env.ConfigParameters.Add("printtoconsole", "0");

-                var alice = env.CreateNode();
                 var bob = env.CreateNode();
                 var miner = env.CreateNode();
+                miner.ConfigParameters.Add("txindex", "1"); // So we can query a tx from txid
                 env.StartAll();
                 Console.WriteLine("Created 3 nodes (alice, bob, miner)");

                 Console.WriteLine("Connect nodes to each other");
-                miner.Sync(alice, true);
                 miner.Sync(bob, true);

                 Console.WriteLine("Generate 101 blocks so miner can spend money");
                 var minerRPC = miner.CreateRPCClient();
                 miner.Generate(101);

-                var aliceRPC = alice.CreateRPCClient();
                 var bobRPC = bob.CreateRPCClient();
                 var bobAddress = bobRPC.GetNewAddress();

                 Console.WriteLine("Alice get money from miner");
-                var aliceAddress = aliceRPC.GetNewAddress();
-                minerRPC.SendToAddress(aliceAddress, Money.Coins(20m));
+                var aliceKey = new Key();
+                var aliceAddress = aliceKey.PubKey.GetAddress(Network.RegTest);
+                var minerToAliceTxId = minerRPC.SendToAddress(aliceAddress, Money.Coins(20m));

                 Console.WriteLine("Mine a block and check that alice is now synched with the miner (same block height)");
                 minerRPC.Generate(1);
-                alice.Sync(miner);
+                
+                var minerToAliceTx = minerRPC.GetRawTransaction(minerToAliceTxId);
+                var aliceUnspentCoins = minerToAliceTx.Outputs.AsCoins()
+                                .Where(c => c.ScriptPubKey == aliceAddress.ScriptPubKey)
+                                .ToDictionary(c => c.Outpoint, c => c);

-                Console.WriteLine($"Alice Balance: {aliceRPC.GetBalance()}");
+                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()}");

                 Console.WriteLine("Alice send 1 BTC to bob");
-                aliceRPC.SendToAddress(bobAddress, Money.Coins(1.0m));
-                Console.WriteLine($"Alice mine her own transaction");
-                aliceRPC.Generate(1);

-                alice.Sync(bob);
+                var txBuilder = Network.RegTest.CreateTransactionBuilder();
+                var aliceToBobTx = txBuilder.AddKeys(aliceKey)
+                         .AddCoins(aliceUnspentCoins.Values)
+                         .Send(bobAddress, Money.Coins(1.0m))
+                         .SetChange(aliceAddress)
+                         .SendFees(Money.Coins(0.00001m))
+                         .BuildTransaction(true);
+
+                Console.WriteLine($"Alice broadcast to miner");
+                minerRPC.SendRawTransaction(aliceToBobTx);
+
+                foreach (var input in aliceToBobTx.Inputs)
+                {
+                    // Let's remove what alice spent
+                    aliceUnspentCoins.Remove(input.PrevOut);
+                }
+                foreach (var output in aliceToBobTx.Outputs.AsCoins())
+                {
+                    if (output.ScriptPubKey == aliceAddress.ScriptPubKey)
+                    {
+                        // Let's add what alice received
+                        aliceUnspentCoins.Add(output.Outpoint, output);
+                    }
+                }
+
+                miner.Generate(1);
+                miner.Sync(bob);

-                Console.WriteLine($"Alice Balance: {aliceRPC.GetBalance()}");
+                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()}");
                 Console.WriteLine($"Bob Balance: {bobRPC.GetBalance()}");
             }
         }
```

The source code is now:

```csharp
using System;
using System.Linq;
using System.Collections.Generic;
using System.Threading;
using NBitcoin;
using NBitcoin.Tests;

namespace NBitcoinTraining
{
    class Program
    {
        static void Main(string[] args)
        {
            // During the first run, this will take time to run, as it download bitcoin core binaries (more than 40MB)
            using (var env = NodeBuilder.Create(NodeDownloadData.Bitcoin.v0_18_0, Network.RegTest))
            {
                // Removing node logs from output
                env.ConfigParameters.Add("printtoconsole", "0");

                var bob = env.CreateNode();
                var miner = env.CreateNode();
                miner.ConfigParameters.Add("txindex", "1"); // So we can query a tx from txid
                env.StartAll();
                Console.WriteLine("Created 3 nodes (alice, bob, miner)");

                Console.WriteLine("Connect nodes to each other");
                miner.Sync(bob, true);

                Console.WriteLine("Generate 101 blocks so miner can spend money");
                var minerRPC = miner.CreateRPCClient();
                miner.Generate(101);
                
                var bobRPC = bob.CreateRPCClient();
                var bobAddress = bobRPC.GetNewAddress();

                Console.WriteLine("Alice get money from miner");
                var aliceKey = new Key();
                var aliceAddress = aliceKey.PubKey.GetAddress(Network.RegTest);
                var minerToAliceTxId = minerRPC.SendToAddress(aliceAddress, Money.Coins(20m));

                Console.WriteLine("Mine a block and check that alice is now synched with the miner (same block height)");
                minerRPC.Generate(1);
                
                var minerToAliceTx = minerRPC.GetRawTransaction(minerToAliceTxId);
                var aliceUnspentCoins = minerToAliceTx.Outputs.AsCoins()
                                .Where(c => c.ScriptPubKey == aliceAddress.ScriptPubKey)
                                .ToDictionary(c => c.Outpoint, c => c);

                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()}");

                Console.WriteLine("Alice send 1 BTC to bob");

                var txBuilder = Network.RegTest.CreateTransactionBuilder();
                var aliceToBobTx = txBuilder.AddKeys(aliceKey)
                         .AddCoins(aliceUnspentCoins.Values)
                         .Send(bobAddress, Money.Coins(1.0m))
                         .SetChange(aliceAddress)
                         .SendFees(Money.Coins(0.00001m))
                         .BuildTransaction(true);

                Console.WriteLine($"Alice broadcast to miner");
                minerRPC.SendRawTransaction(aliceToBobTx);

                foreach (var input in aliceToBobTx.Inputs)
                {
                    // Let's remove what alice spent
                    aliceUnspentCoins.Remove(input.PrevOut);
                }
                foreach (var output in aliceToBobTx.Outputs.AsCoins())
                {
                    if (output.ScriptPubKey == aliceAddress.ScriptPubKey)
                    {
                        // Let's add what alice received
                        aliceUnspentCoins.Add(output.Outpoint, output);
                    }
                }

                miner.Generate(1);
                miner.Sync(bob);

                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()}");
                Console.WriteLine($"Bob Balance: {bobRPC.GetBalance()}");
            }
        }
    }
}
```

The output is basically the same as the previous chapter.

```
Created 3 nodes (alice, bob, miner)
Connect nodes to each other
Generate 101 blocks so miner can spend money
Alice get money from miner
Mine a block and check that alice is now synched with the miner (same block height)
Alice Balance: 20.00000000
Alice send 1 BTC to bob
Alice broadcast to miner
Alice Balance: 18.99999000
Bob Balance: 1.00000000
```

Now this solution is not an improvement over the previous `"Alice sends through Bob through Bitcoin RPC"`. The code is harder to read, and you need to keep track of Alice's unspent coins through `aliceUnspentCoins`, here is some problems:

* What if blocks reorgs?
* What if Alice transaction get malleated or double spent?
* How do your prevent address reuse?

However, can you feel the power of having signed a transaction without relying on any software?

# NBXplorer design

So here are the problem we wish to solve that got problematic in the part `"Alice sends through Bob without Bitcoin RPC"`:
* How to handle block reorgs?
* How to handle malleated transactions or double spent?
* How do your prevent address reuse?

We will use for this a middleware called [NBXplorer](https://github.com/dgarage/NBXplorer/).
The description says:

> A minimalist UTXO tracker for HD Wallets. The goal is to have a flexible, .NET based UTXO tracker for HD wallets. The explorer supports P2SH,P2PKH,P2WPKH,P2WSH and Multi-sig derivation.

It has an easy to use [API](https://github.com/dgarage/NBXplorer/blob/master/docs/API.md), and can run on pruned node.

It is however important to understand how NBXplorer has been designed so you get a better understanding on what you can do with it.
Also, in some case, you might want to do your own UTXO tracker and you may be interested in its design.

NBXplorer just connects to a trusted bitcoin full node. 

Alice will first ask to NBXplorer to track her [HD public key](https://programmingblockchain.gitbook.io/programmingblockchain/key_generation/bip_32).

NBXplorer, will then generate all the scriptPubKey along `0/x`, `1/x` and `x` path, and save them in database.

When NBXplorer receives a transaction (within a block or not), it will check all if any input or output match any scriptPubKey it is tracking.

If it match a tracked scriptPubKey of Alice, then NBXplorer will just add this transaction (along with blockHash if any), to the list of Alice's transactions.

Because NBXplorer has all the transactions of Alice, it can reconstruct the UTXO set of Alice quite easily!

When you query the UTXO set from NBXplorer it retrieves all the transactions from Alice.

If a transaction was spotted in a block, and that this block is not part of the main chain anymore (reorg), NBXplorer will remove this transaction from the list.

From the resulting list of transactions you can easily build this transaction graph:

* A green dot is an output.
* A gray rectangle is a transaction.
* An arrow from an output to a transaction indicate it is getting spent by another transaction.
* Orange arrows indicate a case of double spending.

![](images/UTXO-Construction1.png)

Because the transactions form a acyclic graph, you can create a list which represent the nodes of this graph in a topological ordered way. (If transaction A depends on transaction B, then A will be above B in the list)

But what about the double spending case? Which transaction should be in that list? `17b3b3` or `ab3922`?

The answer is easy: the transaction which arrived last is the most likely to be confirmed eventually. This is because NBXplorer is connected to your trusted full node and your full node would never relay a transaction which conflict with another having more chance to be accepted by miners.

Assuming `ab3922` arrived last, we can make our list.

![](images/UTXO-Construction2.png)


Each time you play a transaction, it makes a modification to the UTXO set. (Read from bottom to the top)

![](images/UTXO-Construction3.png)

So basically, from the transactions, we could calculate that Alice has 4 UTXOs available to spend: `d`,`f`,`j` and `k`.

Designing a wallet tracker in such way make it easy to handle reorgs. It has also the following advantages:

* To restore the UTXO set of an existing wallet, NBXplorer does not have to rescan the whole blockchain, it can just scan Bitcoin's UTXO set. (from Bitcoin's core `scantxoutset`)
* A wallet transaction list is prunable. Notice that if we delete `73bdee` from our database, we will compute exactly the same UTXO set at `ab3922`, so we can handle big wallets.
* No complicated logic to handle reorgs, indexing is insert only.