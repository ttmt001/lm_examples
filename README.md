# lm_examples
Language model examples -- tutorial code for newgrads at TIAL lab.

# Fisher data set
* Original: /g/ssli/projects/disfluencies/fisher
* Various versions split to train and validation: `/g/ssli/projects/disfluencies/ttmt001/fisher_{clean,disf,dtok}`
    * fsh_1* is in valid, the rest in train
* Stats: 
    * train: from 15522 files -- wc >> 1,181,752 sents; 14,363,366 tokens
    * valid: from 3137 files -- wc >> 268,991 sents; 2,841,422 tokens
* Set clean of disfluencies:
    * train: from 15522 files -- wc >>  sents; tokens
    * valid: from 3137 files -- wc >>  sents; tokens

# Steps
Some of these are already done (this is for documentation purposes only)
1. Split to train/valid:
Raw data: mv fisher/text/fsh_1* fisher_disf/valid/
          mv fisher/text/* fisher_disf/train/

Clean data: mv fisher/cleaned/fsh_1* fisher/cleaned/valid/
            mv fisher/cleaned/* fisher/cleaned/train/ 


2. Preprocessings:
    2a0. (if files has associcated features -- dtok set):
    `./grep_words.sh {train,valid}`

    2a1. (clean and dtok set): merge words into sentences; this takes individual files from `fisher/cleaned/{train,valid}` and puts them in `fisher/fisher_clean/{train,valid}`
    `./merge_lines.sh {train,valid}`

    2b. make big text file for ngram models (in both versions)
    `cat train/* > train.txt`
    `cat valid/* > valid.txt`

    2c. change "s" to "'s" in train.txt and valid.txt (clean and disf, not in dtok)
    ```
    %s/\<s\>/'s/g
    %s/\<ll\>/'ll/g
    %s/\<m\>/'m/g
    %s/\<ve\>/'ve/g
    %s/\<d\>/'d/g
    %s/\<re\>/'re/g
    ```
    At this point only train.txt and valid.txt have this fix

3. Make vocabulary from train.txt files:
`python make_vocab.py --step make_vocab --dtype {disf,clean,dtok}`

4. Need this for LSTM model only: split train.txt and valid.txt into smaller chunks to facilitate parallelization:
```
split -d -n 40 train.txt
split -d -n 10 valid.txt
```

5. train ngrams
`./ngram-lms.sh {disf,clean,dtok}`

6. Prepare switchboard (or other dataset) sentences to compute ppl score:
`python prep_lm_sentences.py`

This produces swbd_sents.tsv with turn, sent_num etc. info and ptb as well as ms
versions of the sentences

    6b. For ngram score computations -- produce text files one sent per line
    ```
    cut -f5 swbd_sents.tsv > swbd_ms_sents.txt
    cut -f6 swbd_sents.tsv > swbd_ptb_sents.txt
    ```
    Then remove header line
    OR
    ```
    cut -f5 swbd_sents_with_ann_notok.tsv > swbd_ms_sents_notok.txt
    cut -f6 swbd_sents_with_ann_notok.tsv > swbd_ptb_sents_notok.txt
    ```

7. Compute ngram scores:
`ngrams/ngram-eval.sh {disf,clean,dtok} {ms,ptb}`

8. Convert OOV tokens to <unk> -- preparation step for LSTM LM models:
```
python lstm_lm/make_vocab.py \
   --train_file {x01,..} \
   --dtype {disf,clean,dtok} \
   --unk_split {train,valid}_splits \
   --step prep_data
```

Batching this:
`./run_make_vocab.sh {valid,train} {clean,disf,dtok}`
Then `cat x*_with_unk files > {train,valid}_with_unk.txt`

9. Prepare bucketed data for training LSTM LMs:
NOTE: need to add special words to both clean and disf fisher vocabs: 
`<eos>, <sos>, <pad>`

`python lstm_lm/data_preprocess.py --dtype {disf,clean,dtok} --split {valid,train}`

10. Train LSTM LM on fisher and score on SWBD:
`lstm_lm/job{5000,5001,5002}.sh`

11. Make table of scores (optional):
```
lstm_lm/run_eval_lstm.sh disf 5000 {ms,ptb}
lstm_lm/run_eval_lstm.sh clean 5001 {ms,ptb}
lstm_lm/run_eval_lstm.sh dtok 5002 {ms,ptb}
```

