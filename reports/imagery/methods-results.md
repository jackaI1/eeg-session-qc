# Rest and imagined movement: methods and results

I screened 60 public EEGBCI imagery runs from 20 people using the existing recording quality rules. Runs marked not usable were excluded. The analysis retained 6 people and 240 epochs (rest: 120, imagery: 120). See results.json for each run's decision and source SHA-256.

I filtered each retained recording from 8 to 30 Hz, then extracted 0.5 to 2.5 seconds after each T0 (rest), T1 or T2 (imagery) annotation. Welch power in 8-13 and 13-30 Hz for every channel was log transformed. A standard scaler and regularized logistic regression were fitted inside each training fold. Within-person scores use three stratified folds for each person; held-out scores train on all other people. I report balanced accuracy (the average of the two class recalls), averaged equally across people. The 95% percentile intervals resample people 2,000 times with seed 42; the difference uses paired resampling.

Within-person: 0.631 (95% interval 0.564 to 0.706). Held-out: 0.597 (95% interval 0.536 to 0.675). Difference: 0.033 (95% interval -0.033 to 0.100).

The comparison changes both training population and training size, so the difference is not a pure estimate of leakage. Epochs from the same run can share noise and context. Filtering a whole run before splitting can also share information across nearby epochs. The quality rules are not expert labels; exclusion may bias the sample. These public recordings are not representative of every device or person. This is research, not a diagnostic tool.
