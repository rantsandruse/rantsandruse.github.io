---
layout: post
title:  "Pytorch with NLP in five days: Day 2 - Train an LSTM model in minibatch"
subtitle:  "With proper initialization and padding"
date:   2021-02-21 11:17:18 -0800
categories: pytorch in ten days
---
In day 1 tutorial, we've learned how to work with a very simple LSTM network, by training the model on a single batch of toy data over 
multiple epochs. In this tutorial, I will show you how to train an LSTM model in minibatches, with proper 
variable initialization and padding. 

## Why do we need to train in mini-batches?   
Deep neural networks are data hungry and need to be trained with a large volume of data. 
If you were to train your deep neural network in a single batch, there are a couple of problems: 
1. All of your data may not fit in memory. You will likely run into an out-of-memory error. 
2. Even if you have all the memory you want and will never run into OOM error, training all data in a single batch 
   is not ideal. Prior research showed that a large mini-batch based training routine oftentimes lead 
   to a sharp minima and poor generalization ([Neskar et al](https://arxiv.org/pdf/1609.04836.pdf)). 
   
This practice of training in mini-batches is not just applicable to deep neural networks, but for any model((e.g. SVM, 
random forest & GBM) with stochastic gradient descent based implementation.  

## How do we break down data into minibatches for training etc.?  
   First, we will break our data into three parts: training, validation and test in *train_val_test_split* (link): 
   Here I choose to use *kFold* from *sklearn.model_selection*. (Note that *sklearn.model_selection.stratefiedKFold* cannot be used here, as our target is a sequence instead of a 
   discrete number, making stratification non-trivial):   

    def train_val_test_split(X, X_lens, y, train_val_split = 10, trainval_test_split = 10):
        splits = KFold(n_splits=trainval_test_split, shuffle=True, random_state=42)
        for trainval_idx, test_idx in splits.split(X, y):
            X_trainval, X_test = X[trainval_idx], X[test_idx]
            y_trainval, y_test = y[trainval_idx], y[test_idx]
            X_lens_trainval, X_lens_test = X_lens[trainval_idx], X_lens[test_idx]
    
        splits = KFold(n_splits=train_val_split, shuffle=True, random_state=28)
    
        for train_idx, val_idx in splits.split(X_trainval, y_trainval):
            X_train, X_val = X_trainval[train_idx], X_trainval[val_idx]
            y_train, y_val = y_trainval[train_idx], y_trainval[val_idx]
            X_lens_train, X_lens_val = X_lens_trainval[train_idx], X_lens_trainval[val_idx]
    
        train_dataset = TensorDataset(torch.tensor(X_train, dtype = torch.long),
                                      torch.tensor(y_train, dtype=torch.long),
                                      torch.tensor(X_lens_train, dtype=torch.int64))
    
        val_dataset = TensorDataset(torch.tensor(X_val, dtype=torch.long),
                                    torch.tensor(y_val, dtype=torch.long),
                                    torch.tensor(X_lens_val, dtype=torch.int64))
    
        test_dataset = TensorDataset(torch.tensor(X_test, dtype=torch.long),
                                     torch.tensor(y_test, dtype=torch.long),
                                     torch.tensor(X_lens_test, dtype=torch.int64))
    
        return train_dataset, val_dataset, test_dataset
        
   And then we separate the training data and validation data into small batches using pytorch's *torch.utils.data.DataLoader* in train.py: 
   
        train_loader = DataLoader(dataset = train_dataset, batch_size = batch_size)
        val_loader = DataLoader(dataset = test_dataset, batch_size = batch_size)
   
   (Note: you could also try *BucketIterator* from *torchtext*.)

   We also need to monitor our validation vs. training loss over time to make sure that we are not decreasing training loss 
   at the cost of validation loss: 
        
        for epoch in range(n_epochs):
            train_losses = []
            val_losses = []
            for X_batch, y_batch, X_lens_batch in train_loader:
                optimizer.zero_grad()
                ypred_batch = model(X_batch, X_lens_batch)

                # flatten y_batch and ypred_batch
                loss = loss_fn(ypred_batch.view(batch_size*seq_len, -1), y_batch.view(batch_size * seq_len))
                loss.backward()
                optimizer.step()

                train_losses.append(loss.item())

        with torch.no_grad():
            for X_val, y_val, X_lens_val in val_loader:
                ypred_val = model(X_val, X_lens_val)

                # flatten first
                val_loss = loss_fn(ypred_val.view(batch_size*seq_len, -1), y_val.view(batch_size * seq_len))
                val_losses.append(val_loss.item())

        epoch_train_losses.append(np.mean(train_losses))
        epoch_val_losses.append(np.mean(val_losses))

After training is done, we can visualize our training vs. validation loss in the following:![plot](/assets/day2_img/train_vs_val_loss.png): 

## How do we initialize hidden state? 
   In [tutorial 1](https://github.com/rantsandruse/pytorch_lstm_01intro/blob/main/README.md), you may have noticed that we 
   did not provide input to the initial hidden state of our LSTM network (see main_v1.py(https://github.com/rantsandruse/pytorch_lstm_01intro/blob/main/main_v1.py)):

     lstm_out, (h, c) = self.lstm(embeds.view(len(sentence), 1, -1))

   While in this tutorial, we drew the hidden state from a random uniform distribution using *torch.rand*:

    def init_hidden(self):
        '''
        Initiate hidden states.
        '''
        # Shape for hidden state and cell state: num_layers * num_directions, batch, hidden_size
        h_0 = torch.randn(1, self.batch_size, self.hidden_dim)
        c_0 = torch.randn(1, self.batch_size, self.hidden_dim)

        # The Variable API is now semi-deprecated, so we use nn.Parameter instead.
        # Note: For Variable API requires_grad=False by default;
        # For Parameter API requires_grad=True by default.
        h_0 = nn.Parameter(h_0, requires_grad=True)
        c_0 = nn.Parameter(c_0, requires_grad=True)

        return h_0, c_0

And then we feed it and then feed it into our LSTM network: 

    hidden_0 = self.init_hidden()
    embeds = self.word_embeddings(sentences)
    ...     
    lstm_out, _ = self.lstm(embeds, hidden_0)

   At this point you might ask:

   **What was the initial hidden state for our LSTM network in [tutorial 1](https://github.com/rantsandruse/pytorch_lstm_01intro/blob/main/README.md)? I don't remember parsing it in...**

   This has a simple answer: If you don't parse in hidden state explicitly, it is set to zero by default. 
      
   **Shall we initialize our hidden state randomly or simply set them to zeros**?

   This may not have a simple answer. In general, there are three ways to initialize the hidden state 
   of your LSTM (or RNN network): zero initialization, random initialization, train the initial hidden state as a variable, 
   or some combination of these three options. Each of these methods have its pros and cons. This [blog post](https://r2rt.com/non-zero-initial-states-for-recurrent-neural-networks.html) 
   written by Silviu Pitis provides an excellent explanation (plus experiments) on these options, and I will provide a TL;DR: 
      
   a. Zero initialization is the default initialization method provided by *torch.nn.LSTM*, and it is usually good enough for 
      seq2seq tasks. This initial zero state is arbitrary, but as the network propagates over a long sequence, the impact of this 
      arbitrary initial state is mitigated over steps and almost eliminated by the end. However, zero initialization may not be a good idea if 
      the training text contains many short sentences. As the ratio of state resets to total samples increase, the model 
      becomes increasing tuned to zero state, which leads to overfitting.

   b. Random initialization is oftentimes recommended, to combat the aforementioned the overfitting problem. The additional noise introduced by random 
      initialization makes the model less sensitive to the initialization and thus less likely to overfit. Note that it can also be combined with the next method. 

   c. Learn a default initial hidden state: If we have many samples requiring a state reset for each of them, such as 
      in a sentiment analysis/sequence classification problem, it makes sense to learn a default initial state. But
      if there are only a few long sequences with a handful of state resets, then learning a default state is prone to overfitting as well.

   In our case, we are working with a tiny toy dataset, so it doesn't matter which initialization we use. But ultimately we want to 
   build a sentiment classifier for IMDB reviews, where option b or c would be more appropriate. We implemented b in our code: 

        h_0 = torch.randn(1, self.batch_size, self.hidden_dim)
        c_0 = torch.randn(1, self.batch_size, self.hidden_dim)   

   I will leave it out as an exercise to implement method c, i.e. train your initial hidden state as a model parameter (*Hint: you need to add one or two class parameters in your init function, and remember to set requires_grad=True. The solution is
   provided as comments in the code*). 

   Now that we know when/which initialization method to use, you might ask :
   ***Why should we initialize the hidden state every time we feed in a new batch, instead of once and for all?***    
   Since each of our sample is an independent piece of text data, i.e. we have a lot of "state resets", there's no 
   benefit in memorizing the hidden state from one batch and pass it onto another. That said, if our samples were all part 
   of a long sequence, then memorizing the last hidden state will likely be informative for the next training batch.
   
   Last but not least, we've been discussing the initialization of hidden state, which is **very different** from the initialization of the weights of 
   the LSTM network. For the latter, zero initialization is a very bad idea as you are not "breaking the symmetry". In other words, 
   all of the hidden units are getting the same zero signal. You must use avoid zero initialization for weights and use random 
   initialization or other more advanced methods (e.g. Xavier initialization and Kaimin-He initialization). 
   
### How to pad/pack/mask your sequence and why 
  Pytorch tensors are arrays of uniform length, which means that we need to pad all of our sequences to the same length. 
  But padding your sentence without proper downstream processing could have unintended consequences: 
  
  Imagine that you have a training dataset with 99% of sentences under 10 words, and 1% with 100 words or more. For 99% of the time, 
  your model will try to learn paddings, instead of focusing on the actual sequence with meaningful features.  
  
  This is very inefficient. As your LSTM model will waste most of its time learning hidden states for paddings and not the actual sequence. 
  Besides, since we are training a seq2seq model, if you don't explicitly neglect these sequence paddings, then 
  they will show up in your predictions and creep into your loss function and cause significant bias. 
  For these reasons, you need to do the following: 
  1. Pack your sequence. The padding index is set to 0 by default. 
     ((i.e. Both ground truth and prediction uses tag class 0, 1, 2 for the meaningful classes, and cross entropy loss ignores padding class -1 accordingly): 
     
    embeds = torch.nn.utils.rnn.pack_padded_sequence(embeds, X_lengths, batch_first=True, enforce_sorted=False)

  2. Feed it into your LSTM model
     
    lstm_out, _ = self.lstm(embeds, hidden_0)
  
  Note: we no longer need to reshape the input as we did in tutorial 1.  Since we used the *batch_first=True* option, the 
  input to LSTM here is already (*batch_size, seq_len, hidden_dim*))
      
  3. Pad your sequence back

    lstm_out, _ = torch.nn.utils.rnn.pad_packed_sequence(lstm_out, batch_first=True, total_length = seq_len )
     
  Note: parsing in total_length is a must, otherwise you might run into dimension mismatch.

  4. Last but not least, ask your loss function to ignore the padding classification using ignore_index=0. 
     
     loss_fn = nn.CrossEntropyLoss(ignore_index=0, size_average=True)     
     
  Note: You do not need to remove padding from *output_size*. i.e. Use *len(tag_to_ix)* and not *len(tag_to_ix)-1* when initializing 
     output_size for *LSTMTagger*. 
    
Beware:
 - nn.LSTM function takes in a tensor with the shape (*seq_len, batch_size, hidden_dim*) by default, which is beneficial to tensor operations, but counterintuitive to human users. Switching out 
      batch_first=True allows you parse in a tensor with the shape (*batch_size, seq_len, hidden_dim*). I would recommend the latter to save you a lot of reshaping trouble when parsing mini-batches.
 - *nn.Embedding* Also uses padding_idx=0 by default so there's not need to explicitly set it. Pytorch does NOT accommodate negative padding indices. If you use *padding_idx = -1* with *vocab_size = 5*, then 
     *padding_idx* will become *vocab_size-padding_idx = 4*. It's better to stick to *padding_idx=0*.
  
     
## Further Reading 
 1. [What this tutorial was originally based on, including a few fixes/patches discussed above](https://towardsdatascience.com/taming-lstms-variable-sized-mini-batches-and-why-pytorch-is-good-for-your-health-61d35642972e) 
 2. [On large-batch training for deep learning: generalization gap and sharp minima](https://arxiv.org/pdf/1609.04836.pdf)
 3. [How to create minibatches](https://github.com/chrisvdweth/ml-toolkit/blob/master/pytorch/notebooks/minimal-example-lstm-input.ipynb) 
 3. [How to pad/pack sequence](https://suzyahyah.github.io/pytorch/2019/07/01/DataLoader-Pad-Pack-Sequence.html)
 4. [How to mask/ignore index in cross entropy loss](https://discuss.pytorch.org/t/ignore-index-in-the-cross-entropy-loss/25006/6)
 5. [Non-zero initial states for recurrent neural networks](https://r2rt.com/non-zero-initial-states-for-recurrent-neural-networks.html)
 5. [Forecasting with recurrent neural networks: 12 tricks (See Trick 3 for  proper initialization](http://www.scs-europe.net/conf/ecms2015/invited/Contribution_Zimmermann_Grothmann_Tietz.pdf)

## [Next tutorial](https://github.com/rantsandruse/pytorch_lstm_03classifier) 
 

