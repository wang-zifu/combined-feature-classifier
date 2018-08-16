
# Ham, Spam... or Sham

##### Author: Nathan Worsham
##### Created for MSDS692 Data Science Practicum I at Regis University, 2018
##### [YouTube Presentation](https://www.youtube.com/watch?v=VjLM4uZqhFM)

In the dawn of spam filtering there was a standard juxtaposition of ham versus spam, with "ham" meaning good email and "spam" meaning unwanted email. This categorization however, roles up malicious and junk emails together under the "spam" umbrella. Though it is a different branch of spam email, a more malicious sort that often looks like the real thing is called "phishing" or sometimes "spear phishing" when it is extremely targeted.  Phishing is when an attacker or bad actor (note these are fancy security euphemisms for evil) uses an email message in attempt to trick a user to reveal any sort of information--including a username and password--or to get the user to do an action on the actor’s behalf--like wire them money or send them employee W2s, often with links to a fake website that is masquerading to look like a message that is legitimate (Moramarco, n.d.). Where I work at we use a product called KnowBe4 (https://www.knowbe4.com/) which is primarily used for security awareness training (where we phish our own employees) but it also provides our employees with an option in their email to report something they think is a possibly a phishing email. The problem that happens however, is that often employees don't necessarily understand the difference and will report emails that are simply regular spam emails or emails that the employee is part of a mailing list and not someone trying to scam them.  This causes a lot of noise and requires a security resource to look over the email and decide if it should either be ignored or investigated further.

This creates a classic binary classification problem, that may be able to be solved with machine learning and free up human resources. It is the over arching problem this project proposes to see if can be done, though as a secondary portion of the project I also want to know/see if the final model chosen can be run on Splunk. If you see my [Practicum I project](https://github.com/geekusa/nlp-text-analytics), you may gather I am a Splunk fanatic. Though that may be true, I like seeing what new levels I can push it to. In that first practicum I built tools to help facilitate text analytics inside of Splunk and since the emails for this project will require analyzing and cleaning the text, the two practicums go together nicely. Splunk's data pipeline is easy to maintain and a known entity in our organization, so if the model works in Splunk it could be operationalized there, and stand a better chance of it being maintained by other coworkers rather than just me. a

## Data Cleaning
I had a SOC analyst collect and label a sampling of these emails. I have a bit over 5200 emails collected in their raw form--.eml files. Around 4700 of these are in the "ignore" category but many are duplicates as more than one employee will send the same email and I have a little over 500 of the "requires investigation emails”.

This project tried its hardest to live up to the 80/20 rule of data science where supposedly 80% of your time is spent cleaning the data. I had to keep coming back to fix new flaws I found in the data. For starters every email was an attachment to another email--actually to be more correct, a few were not and because of this, those were not usable for this purpose because they lacked the original header information. Later I had to go through each of the investigate emails with a fine toothed comb.

Here I used Jupyter Python notebooks ([found here](https://github.com/geekusa/combined-feature-classifier/tree/master/PYTHON_NOTEBOOKS/CLEANING)) to keep track of my cleaning. Like I mentioned I had to come back several times and redo, so the only notebooks there are the final cleaning, though between cleanings I would index the data into Splunk so that I could analyze it. Later when I ran into a data leakage issue--more on that later--I was able to add some further investigate emails.

#### SA-mailparser_plus
I learned of a great python library called `mail-parser` (Mantuano, 2016/2018; stackoverflow.com, 2015) and like my previous practicum, I decided to build a [Splunk app](https://splunkbase.splunk.com/app/4129/) with that would package a custom command wrapper around this library so it could be used in Splunk. Though I ended up making the command to to so much more than just parse an email. It also brings back all of the features I needed from the email as long as `adv_attrs` is set to True--which it is by default.

#### TA-mailclient
We had been using this app to scrape email boxes. There is another IMAP app on Splunkbase but I seemed to have good luck with this one. However there was a couple peculiar things I learned about this app. 

It re-writes the email. Replaces the MIME boundary with human readable text, reorders the headers, and removes attachments. I needed the removing attachment part but turns out messing with the MIME format too much causes mail-parser not to work. So I ended up creating my own Python script to extract the inner email (as seen in the cleaning notebooks), leave the boundaries and headers alone (including attachment headers)--this allowed mailparser to still fetch the data it needs--and still remove attachments but leave some helpful information in their place including md5 and sha256. Later I plan on forking the TA-mailclient to make use of this functionality so that my Splunk `mailparser` command will still work.

#### Quoted-printable??
In one of the several runs of deleting the indexed data, re-clean it, and reindex it I decided to take a look at what I had been seeing all throughout the emails, mainly “=20” and “=3D”. I learned that these are called quoted printables in MIME formats and used a library to remove them called “quopri” (stackoverflow.com, 2008). Later I realized in ever email where the text portion is it clearly states “Content-Transfer-Encoding: Quoted-printable” which is the hint that they all do this. 

## Data Exploration
In the last practicum as mentioned prior, I built a Splunk app called [NLP Text Analytics](https://splunkbase.splunk.com/app/4066/), and here I used a portion of this app--specifically the _Counts_ dashboard to look at the textual features of the set.
![both_targets_counts](PROJECT_FILES/IMG/both_targets_counts.png)

The set is heavily dominated by the ignore category and will need to be dealt with in the machine learning phase. The top terms for ignore are nearly identical to the top terms of the set.
![ignore_top_terms](PROJECT_FILES/IMG/ignore_top_terms.png)

However it is interesting to see the different top terms for each category. While the _ignore_ category seems to be dominated by advertising, we can see some of the suspicious nature for the _investigate_ category by it's top terms like "invoice", "payment", "send" and "file".
![investigate_top_terms](PROJECT_FILES/IMG/investigate_top_terms.png)

Looking at just _investigate_'s counts we see the average length of the email--Average Terms Per text_plain--have gone way down, suggesting that suspicious emails tend to be shorter. Later, I ended up under-sampling the majority class and then the average terms for _ignore_ actually increase while the unique term count cuts down by about 2/3rds.  The ngrams definitely show a behavior of trying to coax the user into doing something, though the terms "let know" and "please click" were heavy in the whole set as well, suggesting that it is common language to find in these type of emails. 
![investigate_counts](PROJECT_FILES/IMG/investigate_counts.png)

Next I moved onto exploring the data features that were not text. Here is where a SOC analyst may spend a decent amount of time, under the hood of the email able to see the secrets it is hiding. Changing this into numeric features often resulted in the length of something, the quantity of something occurring along with the unique quantity of something occurring and where or not an email has some feature or not. I started at this point to under-sample the ignore category so that investigate emails would not be flooded out.

One feature that immediately stood out to me and was a bit surprising at first was the length of the body from the email. 
![body_subject_len](PROJECT_FILES/IMG/body_subject_len.png)

Ignore emails body length where more often substantially longer than investigate emails and a great indicator of class. In fact the inner quartile ranges of the two classes' body lengths did not overlap either. Stopping to think about this it seems to be the human element at play here. It would seem that more often than not those behind doing something nefarious with email tend to go the easier route and write shorter emails in order to get a user to do something, whereas an ignore email which often is a marketing email, is filled to the brim with html formatting to make their emails try to stand out as much as possible. This professional html formatting really pushes the length of the email. Subject length here also shows a similar case but not nearly as such a clear divider.

Another interesting though not really surprising find is that investigate emails are more likely to have an attachment. This visualization is in percentage (like a pie chart--the black sheep of statistical visualizations).
![attach_percent](PROJECT_FILES/IMG/attach_percent.png)

Unsurprising because often to accomplish an attack requires a user to execute code, though this can be done with a link, which then downloads and executes the code as well this is a good'ol standby.

TLD which stands for Top Level Domain, in this case is the domain that the from address is using. Looking further into this aspect, the TLD is heavily dominated by ".com" but using a different domain is a common way to spoof or masquerade as an official address. Though when seeing this visually can comparing the two classes, we see that ignore emails are likely to have come from a `.com` address, whereas investigate emails may come from a much larger variety of TLDs.
![treemap_small](PROJECT_FILES/IMG/treemap_small.png)

Though what is also interesting here is that `.net` and `.org` swap places of importance between the two categories and `.net` in general, has greater prominence in the _investigate_ emails.

## Problems Encountered 

One problem I did not quite uncover during EDA but showed up during ML modeling was a data leakage problem. When a large scale phishing campaign is done, often the phisher is going to send very similar or the exact same emails to many recipients. If more than one user alerts on the same email and both emails make it into the dataset, specifically one in the training set and one in the testing set, then the model has an unfair advantage at predicting the email that it really has already seen. On the flip side however, I also found emails with exact same or very similar subjects that clearly were apart of the same campaign but a much different story inside of them. Here is an example of one of these types (using the unix `diff -y` command to compare the emails side-by-side):
![similar_but_different](PROJECT_FILES/IMG/similar_but_different.png)

Here we see the emails are similar but yet they are different enough to both be included in the dataset considering they have enough differentiating attributes--like text body, from TLD, even day of the week and time of day. Why do bad actors do this? My intuition is that they are trying to evade signature detection as probably their first objective, but an auxiliary reason might be to evade human detection in case co-recipients talk and compare notes. Regardless I went through each email that had the exact same subject line or a very similar subject line in order to remove true duplicates which was time consuming. 

Another issue that I had to deal with was an imbalanced dataset which was mentioned earlier in the EDA section. There are plenty of real-world problems in which the event of interest is an uncommon one such as credit card fraud (Chawla, Bowyer, Hall, & Kegelmeyer, 2002). However to get a model to be able to accurately detect the event and not be influenced too heavily by the negative class, over-sampling or under-sampling must take place to strike a balance. I tried using a combination of under-sampling the majority class and SMOTE or Synthetic Minority Over-sampling Technique on the positive class (Chawla, et al., 2002). However, I received better results by just using under-sampling of the majority class, granted this did cause a much smaller dataset than I would have liked.

## ML Modeling
I started the modeling by just working with the non-text features to see how well the predictive power stood of those features by themselves. Earlier in the EDA section I showed non-text features like body and subject length, along with the sender's TLD but I also created many other numerical attributes such as number of links (meaning HTML `<a>` tags) and number of email addresses found. 

One benefit of the much smaller dataset is that it did leave the potential for trying a broad spectrum of algorithms due to the small amount of time involved to crunch numbers. Though of course if the set was bigger, one could always do the same with just a sample of the larger set, if fact that is a good practice to be in the habit of. I ended up creating separate Jupyter notebooks for each of my trials ([found here](https://github.com/geekusa/combined-feature-classifier/tree/master/PYTHON_NOTEBOOKS/FEATURE_TRIALS)). Nonetheless, I ended up with a lot of notebooks because of this, but I found it was easy to copy one and then make some tweaks and then set it aside and then come back to it later when I needed to reference it. This did however cause issues when I wanted to make changes to the existing code, which meant either going back and fixing it in multiple locations or allowing it to drift. 

#### Algorithms Tried

The following is the list of algorithms I tried with each set--though there were a couple of algorithms that simply don't work with a certain feature sets (such as MultinomialNB, as will not take a negative input value).  
```
LinearSVC
SVC with RBF
ExtraTreesClassifier
GradientBoostingClassifier
XGBClassifier
MLPClassifier
MultinomialNB
GaussianNB
RandomForestClassifier
KNeighborsClassifier
NearestCentroid
AdaBoostClassifier
QuadraticDiscriminantAnalysis
GaussianProcessClassifier
DecisionTreeClassifier
LogisticRegression
BaggingClassifierPassiveAggressiveClassifier
RidgeClassifier
SGDClassifier
CalibratedClassifierCV

```
I created a function to collect the scores from the algorithm's performance and then later sorted the output and showed the top 3 for that particular feature set on a particular measure--accuracy, F1, and number of false-negatives.

#### Initial Feature Sets

The different feature sets I tried were:

* Non-Text Features Dense
    * I say dense here because I left continuous variables alone (like body length) and then did one-hot encoding for categorical values. Of all of the tests I ran, this feature set performed the worst but still predicting with over 90% accuracy.

* Non-Text Features Binary
    * Here I did not leave continuous variables alone and cut them into ranges and then did one-hot encoding for them and the categorical values so all I was left with was a sparse data frame of 1s and 0s. I used a combination of `cut` or `qcut` from pandas to make the ranges. Pandas `qcut` chops continuous data into quartiles, but this doesn't also work well when there are severe outliers or not much range at all. ![python_cut](PROJECT_FILES/IMG/python_cut.png) This performed much more balanced than the dense non-text features, as more algorithms overall scored well with this feature set than the previous dense features.

* Text Features, Cleaned, Lemmatized and No Stop words TFIDF
    * Going with the traditional sparse matrix that `TFIDFVectorizer` creates, though I elected to use my own cleaning function. In my cleaning function it removed URLs, email addresses, punctuation and numbers. Next, the set was also lemmatized using the `spaCy` library and stop words removed (using the `spaCy` list as well). I chose to use an ngram range of 1-3 for this feature set.

* Text Features, Cleaned, Lemmatized and No Stop words TF Binary
    * Similar to the set just mentioned, ngrams still set at 1-3, but here I turned off the option of `idf` (or Inverse Document Frequency), `norm` is sent to `None` and set the `binary` option to true. This set just accounts if a word is in the document or not and creates a simple DTM or Document Term Matrix. When I first started trying to fulfill this idea, I neglected to set `norm` to `None`, leaving a default value of `l2` then. Though this still causes binary behavior at one point in the process, the output still remains normalized and similar looking to the previous set. Once I figured out my folly, I then created notebooks that would reference "CompleteBinary" as the set truly and finally was--unfortunately this was one of the things I didn't notice until I was trying to move into Splunk so it is very pervasive. 

* Text Features, Cleaned, Lemmatized and Keep Stop words TFIDF
    * Similar to the first text set but leaving stop words included and because of this I changed the ngram range to 1-4. This set naturally had a lot of features because of this, but the predictive power did not see any impressive increases, so it did not seem worth leaving in stop words for this problem.

* Text Features, Cleaned, Lemmatized and No Stop words TFIDF, LSA
    * Finally, also like the first set but then running the final output through the `TruncatedSVD` algorithm to accomplish LSA or Latent Semantic Analysis. LSA is used in attempt to score words by their meaning in context (Landauer, Foltz, & Laham, 1998). This changes the sparse set into a dense one. I would have thought that this set would outperform the other text sets but I was disappointed, possibly an issue here however is that LSA may perform better on a larger set.

#### Combined Features 
I had trouble finding solid examples of the best way to go about combining text with non-text features. In the end I settled on the concept that in order for it to work, I would either need to make the non-text features sparse, or I would have to make text features dense in order to add them together (stackoverflow.com, 2018). This is so that the dense features do not dominate the sparse features if I were just to combine them as is. I did see and try one idea to have one feature set's prediction be a feature for another set or another idea to have them vote against each other. I moved away from these ideas quickly however because I felt that this would not give them equal footing against each other. Since I was after the concept of modeling this after the mindset that a SOC analyst would take on these emails, I felt it was important for both feature sets to be taken into account so that the whole picture would used to make decision.

* Combined Features Dense
    * This set is a combination of the first non-text set and the LSA set for text features. In order to get the LSA features to merge with the others, I had to change the LSA features back into a data frame.

* Combined Features Spare/Binary
    * This set is a combination of the non-text features changed to binary (second non-text set) with the first text set that is a TFIDF. Again the TFIDF is changed back into a data frame in order to combine. This ended up being one of the final two feature sets chosen to work with.

* Combined Features Binary/Binary
    * The final set of features I chose to work with was likewise a combination of the non-text features changed to binary and the Document Term Matrix binary text set. Like in the previous combination sets the text feature set is changed back into a data frame in order to combine. This was the final of the two feature sets to use.


#### Hyper-parameter Tuning and Final Trials 
I selected two feature sets and three algorithms to explore further because to their performance and stability against this dataset. The two feature sets as just mentioned were "Combined Features Spare/Binary" and "Combined Features Binary/Binary". The three algorithms selected were LinearSVC, ExtraTreesClassifier, and MLPClassifier.

I first worked on tuning the hyper-parameters of the algorithms with the feature sets ([found here](https://github.com/geekusa/combined-feature-classifier/tree/master/PYTHON_NOTEBOOKS/HYPERPARAMETER_TUNING)) using GridSearchCV. Here I ran into two different problems both around the problem of grid search completely hanging. However each issue was different. 

The first issue was with LinearSVC, I could make heads or tails out of it because the algorithm itself completed in seconds. After some digging I was able to find a [Github issue](https://github.com/scikit-learn/scikit-learn/issues/8918) that stated that `return_train_score` is by default set to True and could cause this. Changing this to False fixed the problem and sure enough the documentation talks about performance of the command with plans to change its default in the future. 

The second issue was with MLPClassifier with the same behavior but I already had set `return_train_score=False`. Digging again, after a combination of turning on debugging on gridsearch and the algorithm and this [Github issue](https://github.com/scikit-learn/scikit-learn/issues/5115) tipping me off, I found that the problem was with multi-threading. After setting `n_jobs` to 1, I could finally get GridSearchCV to complete.

Now that I had tuned hyper-parameters, I tried multiple trials with each algorithm individually and the VotingClassifier made up of the tuned algorithms([found here](https://github.com/geekusa/combined-feature-classifier/tree/master/PYTHON_NOTEBOOKS/FINAL_TRIALS)). I then finished by making a voting classifier made up of all three and running cross-validation on it. I had intended on doing such really in the first place, which was why I chose both an odd number of algorithms (so I could have a majority) and why I chose three algorithms that were very different from each other. I then ran 10-fold cross-validation. Looking at their cv results:

_Combined Features Spare/Binary Cross-Validation_
![sparse_binary_cv](PROJECT_FILES/IMG/sparse_binary_cv.png)

_Combined Features Binary/Binary Cross-Validation_
![binary_binary_cv](PROJECT_FILES/IMG/binary_binary_cv.png)

So I was left ultimately with two very similar performing algorithms. Since I was using a tree classifier however, this allowed me to be able to get the `feature_importances_` for each feature set.

_Combined Features Spare/Binary Feature Importances_
![full_sparse_binary_features](PROJECT_FILES/IMG/full_sparse_binary_features.png)

_Combined Features Binary/Binary Feature Importances_
![full_binary_binary_features](PROJECT_FILES/IMG/full_binary_binary_features.png)

The important issue I found here was that the binary/binary model was taking into account much more terms in its top feature importances than the sparse/binary set--specifically 10 vs 1. This was the behavior I was after in a model--which I have mentioned multiple times--one that would give equal significance to both set of features.

Furthermore I had one other reason to select the binary/binary set. In security, we are often more concerned with false-negative performance than with false-positive performance. Granted false-positives lead to alert fatigue but a false-negative in this case would be that the algorithm classified an _investigate_ email as an _ignore_ class--meaning that we would not see these at all. Looking at the false-negative performance using a confusion matrix:

_Combined Features Spare/Binary Confusion Matrix_
![sparse_binary_cm](PROJECT_FILES/IMG/sparse_binary_cm.png)

_Combined Features Binary/Binary Confusion Matrix_
![binary_binary_cm](PROJECT_FILES/IMG/binary_binary_cm.png)

Here we see that the binary/binary feature set has better false-negative performance--one less. Taking these two reasons into account, I now had a clear winner in my mind, the combined binary/binary feature set.

## Move to Splunk

Now that I had the model (or rather models since we are talking about voting) I wanted to use, the final portion of the project was that I wanted to see if I could get the same algorithms working in Splunk. Recently, Splunk upgraded both their underlying scientific Python app and MLTK app, this worked out in my favor as this was the only way I was going to be able to use the MLPClassifier which had a dependency on the version of sklearn. Though now that Splunk's MLTK had MLPClassifier built-in it still did not have several algorithms I needed to complete the move. Fortunately, in the last Practicum, I had gained experience with doing this so I ended up adding four more algorithms to the app ([NLP Text Analytics Splunk App](https://splunkbase.splunk.com/app/4066/)) I had made in that Practicum as in my mind they are related and seemed like as good as place as any for home for the algorithms. 

I added the algorithms LinearSVC, ExtraTreesClassifier, and MinMaxScaler. The last algorithm I needed already existed in Splunk's MLTK which was TfidfVectorizer, but as of version 3.4, it was missing two options that would allow a binary output. Because of this I added a customized version of TfidfVectorizer, called TFBinary, that allowed and already set these options to force binary output.

I then provided the Splunk algorithms the same tuned hyperparamters that I gave Python.
![splunk_fit](PROJECT_FILES/IMG/splunk_fit.png)

I also tried adding the `VotingClassifier` algorithm, however this algorithm takes as arguments is other algorithms. I tried various ways (syntax) of trying to do this, but gave up and chalking it up to the fact that Splunk's MLTK's API does specific formatting to arguments provided to options and currently is not possible. Though probably also a reason I was quick to give up was that I knew I could just implement a hard vote (meaning majority wins) using a Splunk `eval` statement (VotingClassifier also offers a "soft" voting option).
![vote](PROJECT_FILES/IMG/vote.png)

One unsettling thing did happen, which was after doing my 80/20 split in Splunk, training my models, and then pitting them against the 20 percent test set, it kept doing perfect predictions.
![splunk_perfect](PROJECT_FILES/IMG/splunk_perfect.png)

I repeated this exercise several times with different seeds and still received the same results. Though I am not 100% sure what is going on here, one thing I do know is that MLTK is really an abstraction layer to Python, therefore there are going to be some changes and settings going on under the hood that do not exactly match what I was doing in my notebooks. 

However, as a sanity check, I took 400 of the ignore emails that the models had not seen--since I had a large stash of those lying around. 
![splunk_ignore_unseen](PROJECT_FILES/IMG/splunk_ignore_unseen.png)

Seeing these results, the performance was much more in line with what I would expect. Of course the incorrect classifications in this case would be false-positives but as that was all I had I took it as a sign I was on the right course. 

It was interesting doing a comparison of how the Python workflow compared to that of Splunk. They contain essentially the same steps, but the order moves around a bit. Just in Splunk all of the steps become a single search.

## Conclusion

I found that combining text and non-text features would provide a better overall model with this data set. In this way model itself after the workflow of a SOC analyst, taking into account all aspects of an email in question. I was pleased with the performance of using machine learning against this problem, with over 90% accuracy and good false-negative response, I feel like it can start providing a benefit right away in production. I was equally pleased with the experience of moving the models into Splunk. As mentioned, I can then take advantage of Splunk's pipeline and sharing the "search" with others who are not familiar with Machine Learning or even Python might be more inclined to understand what is going on.

## Future Work
As more data comes in over time I will want to continue to improve the model and provide a way for our SOC to continue to train the model. I feel there is room to bring in further features into the model. I did not deal with multi-valued fields in this model as the SA-mailparser_plus app I created for example will return a multivalued field with all of the URL lengths. 

## References
See https://github.com/geekusa/combined-feature-classifier/blob/master/PROJECT_FILES/References.md for the list of references for this paper and project.
