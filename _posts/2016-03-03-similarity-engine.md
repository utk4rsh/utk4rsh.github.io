---
layout: post
title: Simple Similarity Engine
---

This is an attempt to build a simple recommendation engine which recommends similar brands based on provided data set of random shoppers and the brands that they have marked as favorites. The engine takes a brand as input, and recommends 15 similar brands.The dataset is fairly simple and looks like the following


The dataset is fairly simple and looks like the following 

|   UserID   |   ItemID  |   ItemName        |
| ---------- |:---------:| -----------------:|
|     1      |    2      |    Newport News   |
|     1      |    12     |    Aldo           |
|     2      |    41     |    Derek Lam      |
|     2      |     4     |    Moschino       |

For example, the engine might recommend

1. ["Citizen", "Tag Heuer", "Maurice Lacroix", ...] for "Bulova"
2. ["American Eagle", "Aeropostale", "Wet Seal", ... ] for "Hollister"

Implementation
-------------

If we look at the data, the data is fairly simple and features on which recommendation for similar brands can be made should mostly be based on correlation of items with users.

Since we don't know have much information like category of brands, strength of likeness of brand by user, simple similarity computation on item vectors (whose dimensions are the userids) should give a sense of similarity.

There are various vector similarity alogrithms that can be used. The following have be used and implemented

1. CosineSimilarity
2. EuclideanSimilarity
3. PearsonCorellationSimilarity

Since there is no test data, it is hard to say which similarity algorithm is best in this scenario. However it is usually observed that CosineSimilarity performs better.

Steps to Execute
-------------

This is maven project, simple mvn exec can be used to run the project and get brands similar to a known brand.

The command expects 4 paramters : 

1. Similarity Algorithm  to be used. Please check the supported similarity algorithms in description above.
2. Since Data Size is large and execution might consume much memory, similarity can be limited by number of users that should be considered.
3. Due to similar memory reasons, number of brands can also be limited by a certain number
4. Brand ID for which similar items need to recommended.

mvn exec:java -Dexec.mainClass="us.ml.similarity.engine.scratch.ItemSimilarityApp" -Dexec.cleanupDaemonThreads=false -Dexec.args="CosineSimilarity 100000 5000 168"

Sample Output
-------------

Top 15 results for : David Kahn 

|   Item Name                  |   Similarity Weigth    |
| -----------------------------|:----------------------:|
|     Derek Rose               |    0.5773502691896258  |
|     Imagine by Vince Camuto  |    0.5773502691896258  |
|     Jhane Barnes             |    0.5773502691896258  |
|     Magic Suit               |    0.5773502691896258  |
|     Joey New York            |    0.5773502691896258  |
|     Carmen Ho                |    0.5773502691896258  |
|     Da-Nang                  |    0.4714045207910317  |
|     Calphalon                |    0.4082482904638629  |
|     Hilary Radley            |    0.4082482904638629  |
|     Lida Baday               |    0.4082482904638629  |
|     Nic+Zoe                  |    0.4082482904638629  |
|     Tessuto                  |    0.4082482904638629  |
|     Elijah                   |    0.4082482904638629  |
|     Prevata                  |    0.4082482904638629  |
|     Think!                   |    0.4082482904638629  |
|     Sworn Virgins!           |    0.4082482904638629  |

##Thoughts

1. Since there is no strength of likeness or rating from a user on a particular brand, Collaborative filtering commonly used for recommender systems can be used with dummy ratings as well. These techniques however aim to fill in the missing entries of a user-item association matrix. Item-Item or User-User can also be derived once items for a user are filled in the matrix. However CF based on ALS treats the ratings in the user-item matrix as explicit preferences given by the user to the item, I
   could not find any literature if this approach would lead to correct results. ALS in general can be evaluated by computing RMSE (root mean squared error) on existing ratings but the might not be possible for item-item recommendation.

2. Apache Mahout also provides item-item similarity APIs which which uses spark for better memory and compute optimzations.

