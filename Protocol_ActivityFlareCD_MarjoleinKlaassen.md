Project Name: Relating CD actvitiy to the gut microbiome
-------------

Name author of code: Marjolein Klaassen.
Year: 2018

Content:
How to run MaAsLin on Taxonomy data, MetaCyc data, Virulence factors and growth rates 


Setting my working Directory 
-------------
```
setwd("~/Documents/Pilot Project - Virtual Time Line/working directory")
```

Importing clinical metadata databases
-------------
```
db = read.csv("VALFLO.csv", header = T, sep = ";")
db = as.data.frame(db)
VT = read.csv("VIRTUALTIMELINERDEF.csv", header = T, sep = ";")
VT = as.data.frame(VT)
```

 Merging clinical files 
-------------
```
FinalVT = merge (db, VT, by="UMCGNoFromZIC", all = FALSE)
FinalVT=as.data.frame(FinalVT)
```

Only include patients with certain phenotype
-------------
```
FinalVT = FinalVT[FinalVT$IncludedSamples == 'yes',]
FinalVT = FinalVT[,c("Sex", "UMCGIBDDNAID", "PFReads", "AgeAtFecalSampling", "TimeEndPreviousExacerbation", "TimeToStartNextExacerbation", "DiagnosisCurrent", "DiseaseLocation", "MedicationPPI", "AntibioticsWithin3MonthsPriorToSampling", "BMI")]

FinalVT = FinalVT[,c(2, 1, 3, 7, 4, 5, 6, 11, 8, 9, 10)]
```


Taxonomy 
-------------

**Importing Taxonomy Data Metagenomics** 
```
Taxa = read.table ("IBD_brakenCompar.txt", header = TRUE)
```

**Only keeping in species data (grep function)**
```
Taxa = Taxa[-1,]
rownames(Taxa) = Taxa$SID
Taxa = Taxa[,c(2:433)]
temp1 = Taxa[grep("s__", row.names(Taxa)),]
Taxa = as.data.frame(t(temp1))
Taxa2 <- as.data.frame(apply(Taxa, MARGIN = 2, FUN = function(x) as.numeric(as.character(x))))
row.names(Taxa2) = row.names(Taxa)
```

**Making relative abundances of species between 0 and 1 **
```
Taxafinal = Taxa2/100
rowSums(Taxafinal)
```

**Merging clinical data with microbiome data** 
```
Taxafinal["UMCGIBDDNAID"] = row.names(Taxafinal)
Taxafinal=Taxafinal[,c(1237, 1:1236)] 
TaxaVT = merge (FinalVT, Taxafinal, by = "UMCGIBDDNAID", all = FALSE)
```

**Only including CD patients**
```
TaxaVT = TaxaVT[TaxaVT$DiagnosisCurrent== "CD",]
```

**Creating a loop to transfer all days that patients are in a flare, to the numeric value 0**
```
TaxaVT = cbind(TaxaVT[,1:6], "TimePrevVT"=NA, "TimeToStartNextExacerbation"=TaxaVT$TimeToStartNextExacerbation, "TimeNextNegtoZer"=NA, TaxaVT[,8:ncol(TaxaVT)])

TaxaVT$TimeEndPreviousExacerbation = as.numeric(as.character(TaxaVT$TimeEndPreviousExacerbation))
TaxaVT$TimeToStartNextExacerbation = as.numeric(as.character(TaxaVT$TimeToStartNextExacerbation))

for (i in 1:nrow(TaxaVT)) {
  if (TaxaVT$TimeEndPreviousExacerbation[i] < 0 & !is.na(TaxaVT$TimeEndPreviousExacerbation[i])) {
    TaxaVT$TimePrevVT[i] = 0
    TaxaVT$TimeNextNegtoZer[i] = 0
  } else {
    TaxaVT$TimePrevVT[i] = TaxaVT$TimeEndPreviousExacerbation[i]
    TaxaVT$TimeNextNegtoZer[i] = TaxaVT$TimeToStartNextExacerbation[i]
  }
}
```

**Creating a loop to transfer all days 'before an exacerbation' to negative numeric values**
```
TaxaVT = cbind(TaxaVT[,1:9], "TimeNextVT"=NA, TaxaVT[,10:ncol(TaxaVT)])
for (i in 1:nrow(TaxaVT)) {
  if (TaxaVT$TimeNextNegtoZer[i] > 0 & !is.na(TaxaVT$TimeNextNegtoZer[i])) {
    TaxaVT$TimeNextVT[i] = ((TaxaVT$TimeNextNegtoZer[i])*-1)
  }
  else {
    TaxaVT$TimeNextVT[i] = TaxaVT$TimeNextNegtoZer[i]
  }
}
TaxaCD = TaxaVT[,c(1:5, 7, 10, 11:1250)]
```
**When it is not documented whether patient is on PPI, we agreed to report 'no PPI use'**
```
for (i in 1:nrow(TaxaCD)){
  if (is.na(TaxaCD$MedicationPPI[i])){
    TaxaCD$MedicationPPI[i] = "no"
  } else 
    TaxaCD$MedicationPPI[i] = TaxaCD$MedicationPPI[i]
}
```

**When it is not documented whether patient is on antibiotics, we agreed to report 'no antibiotic use'**
```
for (i in 1:nrow(TaxaCD)){
  if (is.na(TaxaCD$AntibioticsWithin3MonthsPriorToSampling[i])){
    TaxaCD$AntibioticsWithin3MonthsPriorToSampling[i] = "no"
  } else 
    TaxaCD$AntibioticsWithin3MonthsPriorToSampling[i] = TaxaCD$AntibioticsWithin3MonthsPriorToSampling[i]
}
```

**Checking whether patients that are in an exacerbation, have both numeric value 0 to the last flare and next flare**
```
for (i in 1:nrow(TaxaCD)){
  if (!is.na(TaxaCD$TimePrevVT[i]) & TaxaCD$TimePrevVT[i] == 0 ){
    TaxaCD$TimeNextVT[i] = 0
  } else 
    TaxaCD$TimeNextVT[i] = TaxaCD$TimeNextVT[i]
}

for (i in 1:nrow(TaxaCD)){
  if (!is.na(TaxaCD$TimeNextVT[i]) & TaxaCD$TimeNextVT[i] == 0 ){
    TaxaCD$TimePrevVT[i] = 0
  } else 
    TaxaCD$TimePrevVT[i] = TaxaCD$TimePrevVT[i]
}
```

**Filtering out all microbiome species who are present in <5% of patients**
```
TaxonomyFilter = TaxaCD[,c(1, 12: 1247)]
TaxonomyFilter2 = TaxonomyFilter[,-1]
rownames(TaxonomyFilter2) = TaxonomyFilter[,1]
TaxonomyFilter2 = as.data.frame(t(TaxonomyFilter2))
TaxonomyFilter2[TaxonomyFilter2==0.00000000] =0.000000000
TaxonomyFilter2[TaxonomyFilter2==0] =0.000000000

TaxonomyFilter2 = TaxonomyFilter2[rowSums(TaxonomyFilter2==0.000000000)<=ncol(TaxonomyFilter2)*0.95,]

TaxonomyFilter2 = as.data.frame(t(TaxonomyFilter2))
TaxonomyFilter2["UMCGIBDDNAID"] = row.names(TaxonomyFilter2)
TaxonomyFilter2=TaxonomyFilter2[,c(301, 1:300)]

TaxaVT = TaxaCD[,c(1:11)]
TaxaVT = merge(TaxaVT, TaxonomyFilter2, by= "UMCGIBDDNAID", all = FALSE)
```

Analyses MaAsLin Taxonomy 
-------------

**Comparing patients in a flare with patients in remission** 

**Removing all patients that have no documented exacerbation** 
```
TaxaInFlareNot = TaxaVT 
TaxaInFlareNot<-TaxaInFlareNot[!with(TaxaInFlareNot,is.na(TaxaInFlareNot$TimeNextVT)& is.na(TaxaInFlareNot$TimePrevVT)),]
```
**Creating a loop to create new phenotype 'in flare/ not in flare'**
```
TaxaInFlareNot = cbind(TaxaInFlareNot[,1:7], "InFlareNot"=NA, TaxaInFlareNot[,8:ncol(TaxaInFlareNot)])
TaxaInFlareNot$InFlareNot = as.numeric(as.character(TaxaInFlareNot$InFlareNot))

for (i in 1:nrow(TaxaInFlareNot)){
  if (is.na(TaxaInFlareNot$TimeNextVT[i]) | is.na(TaxaInFlareNot$TimePrevVT[i])){
    TaxaInFlareNot$InFlareNot[i] = "Not in a flare"
  } else if (TaxaInFlareNot$TimeNextVT[i]== 0 | TaxaInFlareNot$TimePrevVT==0){
    TaxaInFlareNot$InFlareNot[i] = "In a flare"
  } else if (TaxaInFlareNot$TimeNextVT[i] <0 & TaxaInFlareNot$TimePrevVT >0) {
    TaxaInFlareNot$InFlareNot[i] = "Not in a flare"
  } else
    TaxaInFlareNot$InFlareNot[i] = "Not in a flare"
}
```

**Creating order of columns suited for MaAsLin (first patientIDs, then clinical metadata, then microbiome data) **
```
TaxaInFlareNot = TaxaInFlareNot[,c(1, 8, 2, 3, 5, 9:312)]
```

**MaAsLin requires a tsv/csv file as input file of all the data (the R-database you just made)** 
```
write.table(TaxaInFlareNot, "InFlareNot.tsv", sep = "\t", quote = F, row.names = F)
```

**MaAsLin requires the tsv/csv file, the name of the output file in which MaAsLin puts all the results, and the R-script which says which columns/rows he should analyze (i.e. file.read.config). Furthermore, I forced a zero inflated model (fZeroInlfated = T) and set a minimal abundance of the microbiome feature at 25% of samples. I also force correction of phenotypes.**
```
Maaslin('InFlareNot.tsv','nOud Final Taxa Analysis 1',strInputConfig = '1.TaxaInFlare.read.config', fZeroInflated = T, dMinSamp = 0.25, strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'))
```

**Analysis 2: comparison in MaAsLin patients before and patients in an exacerbation **  

```
InFlareNot = TaxaVT
InFlareNot<-InFlareNot[!with(InFlareNot,is.na(InFlareNot$TimeNextVT)& is.na(InFlareNot$TimePrevVT)),]

InFlareNot = cbind(InFlareNot[,1:7], "TempColFlare"=NA, InFlareNot[,8:ncol(InFlareNot)])
for (i in 1:nrow(InFlareNot)){
  if (is.na(InFlareNot$TimeNextVT[i])) {
    InFlareNot$TempColFlare[i]= "None"
    InFlareNot$TimeNextVT[i] = "None"
  } else if (is.na(InFlareNot$TimePrevVT[i])){
    InFlareNot$TimePrevVT[i] = "None"
  } else if (InFlareNot$TimeNextVT[i]< 0) {
    InFlareNot$TempColFlare[i] = "before a flare"
  } else {
    InFlareNot$TempColFlare[i] = "during a flare"
  }
}

for (i in 1:nrow(InFlareNot)){
  if (InFlareNot$TimeNextVT[i] =="None"){
    InFlareNot$TempColFlare[i] = "after a flare"
  } else if (InFlareNot$TimePrevVT[i] =="None"){
    InFlareNot$TempColFlare[i] = "before a flare"
  } else if (InFlareNot$TimeNextVT[i] == 0){
    InFlareNot$TempColFlare[i] = "during a flare"
  } else if ((InFlareNot$TimeNextVT[i] != "None") & (InFlareNot$TimePrevVT[i] != "None")){ 
    if (as.numeric(InFlareNot$TimeNextVT[i]) + as.numeric(InFlareNot$TimePrevVT[i]) > 0){
      InFlareNot$TempColFlare[i] = "before a flare"
    } else {
      InFlareNot$TempColFlare[i] = "after a flare"
    }
  }
}

# For this analysis, I only want to compare before a flare with during a flare. So I remove all samples that 
# are closer to their last flare. 
InFlareNot = InFlareNot[InFlareNot$TempColFlare!= "after a flare",]

InFlareNot = InFlareNot[,c(1, 8, 2, 3, 5, 9:312)]

# creating tsv file for MaAsLin

write.table(InFlareNot, "InFlareNot.tsv", sep = "\t", quote = F, row.names = F)
```

**MaAsLin analysis 2 **
```
Maaslin('InFlareNot.tsv','nOud Final Taxonomy (species) analysis 2',strInputConfig = '2.TaxaInFlare.read.config', dMinSamp = 0.25, fZeroInflated = T,strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'))
```

Analysis MaAslin analysis 3: Categorical comparison of gut metagenome patients duringa flare with all patients after a flare
-------------
```
InFlareNot = TaxaVT

InFlareNot<-InFlareNot[!with(InFlareNot,is.na(InFlareNot$TimeNextVT)& is.na(InFlareNot$TimePrevVT)),]
InFlareNot = cbind(InFlareNot[,1:7], "TempColFlare"=NA, InFlareNot[,8:ncol(InFlareNot)])

for (i in 1:nrow(InFlareNot)){
  if (is.na(InFlareNot$TimeNextVT[i])) {
    InFlareNot$TempColFlare[i]= "None"
    InFlareNot$TimeNextVT[i] = "None"
  } else if (is.na(InFlareNot$TimePrevVT[i])){
    InFlareNot$TimePrevVT[i] = "None"
  } else if (InFlareNot$TimeNextVT[i]< 0) {
    InFlareNot$TempColFlare[i] = "before a flare"
  } else {
    InFlareNot$TempColFlare[i] = "during a flare"
  }
}

for (i in 1:nrow(InFlareNot)){
  if (InFlareNot$TimeNextVT[i] =="None"){
    InFlareNot$TempColFlare[i] = "after a flare"
  } else if (InFlareNot$TimePrevVT[i] =="None"){
    InFlareNot$TempColFlare[i] = "before a flare"
  } else if (InFlareNot$TimeNextVT[i] == 0){
    InFlareNot$TempColFlare[i] = "during a flare"
  } else if ((InFlareNot$TimeNextVT[i] != "None") & (InFlareNot$TimePrevVT[i] != "None")){ 
    if (as.numeric(InFlareNot$TimeNextVT[i]) + as.numeric(InFlareNot$TimePrevVT[i]) > 0){
      InFlareNot$TempColFlare[i] = "before a flare"
    } else {
      InFlareNot$TempColFlare[i] = "after a flare"
    }
  }
}


# Now, I only want to compare in a flare with after a flare, so I remove samples which are closer to their
# next flare. 
InFlareNot = InFlareNot[InFlareNot$TempColFlare!= "before a flare",]


InFlareNot = InFlareNot[,c(1, 8, 2, 3, 5, 9:312)]

# Creating tsv file of database 
write.table(InFlareNot, "InFlareNot.tsv", sep = "\t", quote = F, row.names = F)
```

**MaAsLin analysis 3: between patients >1 year quiescent disease versus in a flare**
```
Maaslin('InFlareNot.tsv','nOud Final Taxonomy (species) analysis 3',strInputConfig = '3.TaxaInFlare.read.config', dMinSamp = 0.25, fZeroInflated = T,strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'))
```

Analysis MaAsLin 4: Linear analysis of patients who have their next flare <6 months - time until next flare
-------------

```
TaxaCDIIa = TaxaVT
TaxaCDIIa<-TaxaCDIIa[!with(TaxaCDIIa,is.na(TaxaCDIIa$TimeNextVT)& is.na(TaxaCDIIa$TimePrevVT)),]

TaxaCDIIa = cbind(TaxaCDIIa[,1:7], "LinBefore"=NA, TaxaCDIIa[,8:ncol(TaxaCDIIa)])
TaxaCDIIa$LinBefore = as.numeric(as.character(TaxaCDIIa$LinBefore))

for (i in 1:nrow(TaxaCDIIa)){
  if (is.na(TaxaCDIIa$TimeNextVT[i])) {
    TaxaCDIIa$LinBefore[i]= "None"
    TaxaCDIIa$TimeNextVT[i] = "None"
  } else if (is.na(TaxaCDIIa$TimePrevVT[i])){
    TaxaCDIIa$TimePrevVT[i] = "None"
  } else if (TaxaCDIIa$TimeNextVT[i]< 0) {
    TaxaCDIIa$LinBefore[i] = "before a flare"
  } else {
    TaxaCDIIa$LinBefore[i] = "during a flare"
  }
}

for (i in 1:nrow(TaxaCDIIa)){
  if (TaxaCDIIa$TimeNextVT[i] =="None"){
    TaxaCDIIa$LinBefore[i] = "after a flare"
  } else if (TaxaCDIIa$TimePrevVT[i] =="None"){
    TaxaCDIIa$LinBefore[i] = "before a flare"
  } else if (TaxaCDIIa$TimeNextVT[i] == 0){
    TaxaCDIIa$LinBefore[i] = "during a flare"
  } else if ((TaxaCDIIa$TimeNextVT[i] != "None") & (TaxaCDIIa$TimePrevVT[i] != "None")){ 
    if (as.numeric(TaxaCDIIa$TimeNextVT[i]) + as.numeric(TaxaCDIIa$TimePrevVT[i]) > 0){
      TaxaCDIIa$LinBefore[i] = "before a flare"
    } else {
      TaxaCDIIa$LinBefore[i] = "after a flare"
    }
  }
}

# For this analysis, I only want to include patients who are in the 6 months before their next flare. Therefore,
# i remove patients who are closer to their last flare or are in a flare. 
TaxaCDIIa = TaxaCDIIa[TaxaCDIIa$LinBefore!= "after a flare",]
TaxaCDIIa = TaxaCDIIa[TaxaCDIIa$LinBefore!= "during a flare",]

# Now, I only want to include patients who have their next flare in the next half year. Therefore, I firstly make the
# days until the next flare numeric again. 
TaxaCDIIa$TimeNextVT = as.numeric(as.character(TaxaCDIIa$TimeNextVT))
# Then,I make all days that are more than 182.5 (half year) days away from their next flare, into NAs.  
TaxaCDIIa$TimeNextVT[TaxaCDIIa$TimeNextVT< (-182.5)]<-NA
#Then, I remove those samples that have NAs (i.e. are further away from their next flare than 6 months), by
# only maintaining samples with complete values, instead of NAs. 
TaxaCDIIa = TaxaCDIIa[!is.na(TaxaCDIIa$TimeNextVT),]
# I remove the column which has the days since the last flare. 
TaxaCDIIa = TaxaCDIIa[,c(1:5, 7:312)]
# I create the order of columns suitable for MaAsLin. 
TaxaCDIIa = TaxaCDIIa[,c(1, 6, 2:5, 8:311)]

# creating TSV file 
write.table(TaxaCDIIa, "LinBeforein1Yr.tsv", sep = "\t", quote = F, row.names = F)
```

**MaAsLin run 2a (Patients who have next flare < 1 year) - (time until next flare) ** 
Maaslin('LinBeforein1Yr.tsv','nOud Taxonomy (species) analysis 4a',strInputConfig = '2a.Taxa.read.config', dMinSamp = 0.25, fZeroInflated = T, strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'))




##20.5. Analyses 2b: (Patients who had last flare < 1 year) - (time since last flare). 
##20.5.a Give this dataframe a new name. 
LinAfterIIb = TaxaVT

##20.5.b I remove patients that have not a documented previous and next flare in the EHR.
LinAfterIIb<-LinAfterIIb[!with(LinAfterIIb,is.na(LinAfterIIb$TimeNextVT)& is.na(LinAfterIIb$TimePrevVT)),]
##20.5.d. I create a new column, so I can give a new value 'before a flare' or 'in a flare' to all samples. 
# When time to next flare is na, the new column is "none" (just so that these samples have a variable).
# Furthermore, when the time til the next flare is < 0, this patient is before a flare. 
LinAfterIIb = cbind(LinAfterIIb[,1:7], "LinAfter"=NA, LinAfterIIb[,8:ncol(LinAfterIIb)])
LinAfterIIb$LinAfter = as.numeric(as.character(LinAfterIIb$LinAfter))

##20.5.e Now, Again I ascribe samples to which flare is closest. 
for (i in 1:nrow(LinAfterIIb)){
  if (is.na(LinAfterIIb$TimeNextVT[i])) {
    LinAfterIIb$LinAfter[i]= "None"
    LinAfterIIb$TimeNextVT[i] = "None"
  } else if (is.na(LinAfterIIb$TimePrevVT[i])){
    LinAfterIIb$TimePrevVT[i] = "None"
  } else if (LinAfterIIb$TimeNextVT[i]< 0) {
    LinAfterIIb$LinAfter[i] = "before a flare"
  } else {
    LinAfterIIb$LinAfter[i] = "during a flare"
  }
}

for (i in 1:nrow(LinAfterIIb)){
  if (LinAfterIIb$TimeNextVT[i] =="None"){
    LinAfterIIb$LinAfter[i] = "after a flare"
  } else if (LinAfterIIb$TimePrevVT[i] =="None"){
    LinAfterIIb$LinAfter[i] = "before a flare"
  } else if (LinAfterIIb$TimeNextVT[i] == 0){
    LinAfterIIb$LinAfter[i] = "during a flare"
  } else if ((LinAfterIIb$TimeNextVT[i] != "None") & (LinAfterIIb$TimePrevVT[i] != "None")){ 
    if (as.numeric(LinAfterIIb$TimeNextVT[i]) + as.numeric(LinAfterIIb$TimePrevVT[i]) > 0){
      LinAfterIIb$LinAfter[i] = "before a flare"
    } else {
      LinAfterIIb$LinAfter[i] = "after a flare"
    }
  }
}

# 20.5.e For this analysis, I only want to include patients who are in the 6 months after their prior flare. Therefore,
# i remove patients who are closer to their next flare or are in a flare. 
LinAfterIIb = LinAfterIIb[LinAfterIIb$LinAfter!= "before a flare",]
LinAfterIIb = LinAfterIIb[LinAfterIIb$LinAfter!= "during a flare",]
# 20.5.f Now, I only want to include patients who have their next flare in the next half year. Therefore, I firstly make the
# days until the next flare numeric again.
LinAfterIIb$TimePrevVT = as.numeric(as.character(LinAfterIIb$TimePrevVT))

# 20.5.g Then,I make all days that are more than 182.5 (half year) days away from their last flare, into NAs.  
LinAfterIIb$TimePrevVT[LinAfterIIb$TimePrevVT> 182.5]<-NA
#20.5.h Then, I remove those samples that have NAs (i.e. are further away from their next flare than 6 months), by
# only maintaining samples with complete values, instead of NAs. 
LinAfterIIb = LinAfterIIb[!is.na(LinAfterIIb$TimePrevVT),]

# 20.5.j I create the order of columns suitable for MaAsLin. 
LinAfterIIb = LinAfterIIb[,c(1, 6, 2, 3, 5, 9:311)]
write.table(LinAfterIIb, "LinAfterin1Yr.tsv", sep = "\t", quote = F, row.names = F)
##2b 
Maaslin('LinAfterin1Yr.tsv','nOud Taxonomy 2b (species) analyses 4b',strInputConfig = '2bTaxa.read.config', dMinSamp = 0.25, fZeroInflated = T, strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'))












































## ____________________________________________________________________________
###Next, I will perform the exact same analyses for functional pathways. 
db = read.csv("VALFLO.csv", header = T, sep = ";")
db = as.data.frame(db)

VT = read.csv("VIRTUALTIMELINERDEF.csv", header = T, sep = ";")
VT = as.data.frame(VT)

FinalVT = merge (db, VT, by="UMCGNoFromZIC", all = FALSE)
FinalVT=as.data.frame(FinalVT)
FinalVT = FinalVT[FinalVT$IncludedSamples == 'yes',]
FinalVT = FinalVT[,c("Sex", "UMCGIBDDNAID", "PFReads", "AgeAtFecalSampling", "TimeEndPreviousExacerbation", "TimeToStartNextExacerbation", "DiagnosisCurrent", "DiseaseLocation", "MedicationPPI", "AntibioticsWithin3MonthsPriorToSampling", "BMI")]
FinalVT = FinalVT[,c(2, 1, 3, 7, 4, 5, 6, 11, 8, 9, 10)]

## Importing Kraken metagenomic sequencing file 'MetaCycpathways'. 
Metacyc = read.delim ("Metacyc_paths.txt", header = TRUE, sep = "\t")
Metacyc = as.data.frame(Metacyc)

## Since MaAsLin requires proportional data, I make the data proportional. 
MetacycProp= data.frame(matrix(nrow=nrow(Metacyc), ncol=ncol(Metacyc)))
colsum = colSums(Metacyc)
for (i in 1:ncol(Metacyc)) {
  for (j in 1:nrow(Metacyc)){MetacycProp[j,i]<- 100*Metacyc[j,i]/colsum[i]}
}

rownames(MetacycProp) = rownames(Metacyc)
colnames(MetacycProp) = colnames(Metacyc)
colSums(MetacycProp)

MetacycProp1 = MetacycProp
## Nu is de data proportional. Nu nog aan de vereiste voldoen om de data tussen
## 0 en 1 aan te leveren. 
MetacycProp1= MetacycProp1/100
colSums(MetacycProp1)

## zorgen dat de kolommen dezelfde naam hebben voor het mergen
MetacycProp1 = as.data.frame(t(MetacycProp1))
MetacycProp1["UMCGIBDDNAID"] = row.names(MetacycProp1)
MetacycProp1=MetacycProp1[,c(784,1:783)]

## Merging the taxonomy file and the metadata file 'time until flare'. 
MetaCycVTFin = merge (FinalVT, MetacycProp1, by= "UMCGIBDDNAID", all = FALSE)


## Converting all negative numbers (patient is in a flare multiple days) into 
## zero's (meaning that all patients in a flare are stated as just 'in a flare' =
## 0 days until last flare and 0 days until next flare)
MetaCycVTFin = cbind(MetaCycVTFin[,1:6], "TimePrevVT"=NA, "TimeToStartNextExacerbation"=MetaCycVTFin$TimeToStartNextExacerbation, "TimeNextNegtoZer"=NA, MetaCycVTFin[,8:ncol(MetaCycVTFin)])

MetaCycVTFin$TimeEndPreviousExacerbation = as.numeric(as.character(MetaCycVTFin$TimeEndPreviousExacerbation))
MetaCycVTFin$TimeToStartNextExacerbation = as.numeric(as.character(MetaCycVTFin$TimeToStartNextExacerbation))

for (i in 1:nrow(MetaCycVTFin)) {
  if (MetaCycVTFin$TimeEndPreviousExacerbation[i] < 0 & !is.na(MetaCycVTFin$TimeEndPreviousExacerbation[i])) {
    MetaCycVTFin$TimePrevVT[i] = 0
    MetaCycVTFin$TimeNextNegtoZer[i] = 0
  } else {
    MetaCycVTFin$TimePrevVT[i] = MetaCycVTFin$TimeEndPreviousExacerbation[i]
    MetaCycVTFin$TimeNextNegtoZer[i] = MetaCycVTFin$TimeToStartNextExacerbation[i]
  }
}

## Making 'days to the next flare' all zero. 
MetaCycVTFin = cbind(MetaCycVTFin[,1:9], "TimeNextVT"=NA, MetaCycVTFin[,10:ncol(MetaCycVTFin)])
for (i in 1:nrow(MetaCycVTFin)) {
  if (MetaCycVTFin$TimeNextNegtoZer[i] > 0 & !is.na(MetaCycVTFin$TimeNextNegtoZer[i])) {
    MetaCycVTFin$TimeNextVT[i] = ((MetaCycVTFin$TimeNextNegtoZer[i])*-1)
  }
  else {
    MetaCycVTFin$TimeNextVT[i] = MetaCycVTFin$TimeNextNegtoZer[i]
  }
}

MetaCycCD = MetaCycVTFin[MetaCycVTFin$DiagnosisCurrent == 'CD',]
MetaCycCD = MetaCycCD[,c(1:5, 7, 10, 11:797)]


# When PPI use is not documented in patient, it is agreed that we report 'no PPI use'.
for (i in 1:nrow(MetaCycCD)){
  if (is.na(MetaCycCD$MedicationPPI[i])){
    MetaCycCD$MedicationPPI[i] = "no"
  } else 
    MetaCycCD$MedicationPPI[i] = MetaCycCD$MedicationPPI[i]
}

# When Antibiotic use is not documented in patient, it is agreed that we report 'no ab use'. 
for (i in 1:nrow(MetaCycCD)){
  if (is.na(MetaCycCD$AntibioticsWithin3MonthsPriorToSampling[i])){
    MetaCycCD$AntibioticsWithin3MonthsPriorToSampling[i] = "no"
  } else 
    MetaCycCD$AntibioticsWithin3MonthsPriorToSampling[i] = MetaCycCD$AntibioticsWithin3MonthsPriorToSampling[i]
}

# Zorgen dat het helemaal klopt, dan alle NextVT die 0 zijn, dan ook PrevVT hebben die nul is.
for (i in 1:nrow(MetaCycCD)){
  if (!is.na(MetaCycCD$TimePrevVT[i]) & MetaCycCD$TimePrevVT[i] == 0 ){
    MetaCycCD$TimeNextVT[i] = 0
  } else 
    MetaCycCD$TimeNextVT[i] = MetaCycCD$TimeNextVT[i]
}

for (i in 1:nrow(MetaCycCD)){
  if (!is.na(MetaCycCD$TimeNextVT[i]) & MetaCycCD$TimeNextVT[i] == 0 ){
    MetaCycCD$TimePrevVT[i] = 0
  } else 
    MetaCycCD$TimePrevVT[i] = MetaCycCD$TimePrevVT[i]
}

MetaCycVT = MetaCycCD

### Filtering out species that are abundant in <5% of patients
MCFilter = MetaCycCD[,c(1, 12: 794)]
MCFilter2 = MCFilter[,-1]
rownames(MCFilter2) = MCFilter[,1]
MCFilter2 = as.data.frame(t(MCFilter2))
MCFilter2 = MCFilter2[rowSums(MCFilter2==0.0000000000)<=ncol(MCFilter2)*0.95,]
MCFilter2 = as.data.frame(t(MCFilter2))
MCFilter2["UMCGIBDDNAID"] = row.names(MCFilter2)
MCFilter2=MCFilter2[,c(634, 1:633)]


MetaCycVT = MetaCycCD[,c(1:11)]
MetaCycVT = merge(MetaCycVT, MCFilter2, by= "UMCGIBDDNAID", all = FALSE)


## Er zijn dubbele punten in de kolomnamen. Hier kan MaAslin niet mee werken
## Nu wil ik dus alle ':' vervangen door '_'.
names(MetaCycVT) = gsub(x = names(MetaCycVT), pattern = ":", replacement = "_") 
names(MetaCycVT) = gsub(x = names(MetaCycVT), pattern = " ", replacement = "_") 
names(MetaCycVT) = gsub(x = names(MetaCycVT), pattern = "-", replacement = "_") 
names(MetaCycVT) = gsub(x = names(MetaCycVT), pattern = ")", replacement = "_") 
names(MetaCycVT) = gsub(x = names(MetaCycVT), pattern = "/", replacement = "_")
MetaCycVTTidy = make.names(colnames(MetaCycVT), unique = TRUE)
colnames(MetaCycVT) = MetaCycVTTidy 




###### Analysis 1: categorical comparison of gut metagenome, all patients in a flare with
###### all patients not in a flare. 

MCInFlareNot = MetaCycVT
## Remove patients that have NA in both Prev/Next <1 yr flare
MCInFlareNot<-MCInFlareNot[!with(MCInFlareNot,is.na(MCInFlareNot$TimeNextVT)& is.na(MCInFlareNot$TimePrevVT)),]

MCInFlareNot = cbind(MCInFlareNot[,1:7], "InFlareNot"=NA, MCInFlareNot[,8:ncol(MCInFlareNot)])
MCInFlareNot$InFlareNot = as.numeric(as.character(MCInFlareNot$InFlareNot))

for (i in 1:nrow(MCInFlareNot)){
  if (is.na(MCInFlareNot$TimeNextVT[i]) | is.na(MCInFlareNot$TimePrevVT[i])){
    MCInFlareNot$InFlareNot[i] = "Not in a flare"
  } else if (MCInFlareNot$TimeNextVT[i]== 0 | MCInFlareNot$TimePrevVT==0){
    MCInFlareNot$InFlareNot[i] = "In a flare"
  } else if (MCInFlareNot$TimeNextVT[i] <0 & MCInFlareNot$TimePrevVT >0) {
    MCInFlareNot$InFlareNot[i] = "Not in a flare"
  } else
    MCInFlareNot$InFlareNot[i] = "Not in a flare"
}


MCInFlareNot = MCInFlareNot[,c(1, 8, 2, 3, 5, 9:645)]
write.table(MCInFlareNot, "InFlareNot.tsv", sep = "\t", quote = F, row.names = F)

## MaAsLin analysis 1: between patients >1 year quiescent disease versus in a flare 
Maaslin('InFlareNot.tsv','nOud Final MetaCyc Analysis 1',strInputConfig = '1.MetaCyc.read.config', fZeroInflated = T,strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'))




###### Analysis 2: categorical comparison of gut metagenome, all patients before a flare 
######  with all patients during a flare 

MCInFlareNot = MetaCycVT
## Remove patients that have NA in both Prev/Next <1 yr flare
MCInFlareNot<-MCInFlareNot[!with(MCInFlareNot,is.na(MCInFlareNot$TimeNextVT)& is.na(MCInFlareNot$TimePrevVT)),]

## Creating new column 'in flare/ not in flare >1 yr'
MCInFlareNot = cbind(MCInFlareNot[,1:7], "TempColFlare"=NA, MCInFlareNot[,8:ncol(MCInFlareNot)])

## Giving value to new column 'in flare/ not in flare >1 yr'
for (i in 1:nrow(MCInFlareNot)){
  if (is.na(MCInFlareNot$TimeNextVT[i])) {
    MCInFlareNot$TempColFlare[i]= "None"
    MCInFlareNot$TimeNextVT[i] = "None"
  } else if (is.na(MCInFlareNot$TimePrevVT[i])){
    MCInFlareNot$TimePrevVT[i] = "None"
  } else if (MCInFlareNot$TimeNextVT[i]< 0) {
    MCInFlareNot$TempColFlare[i] = "before a flare"
  } else {
    MCInFlareNot$TempColFlare[i] = "during a flare"
  }
}
#
for (i in 1:nrow(MCInFlareNot)){
  if (MCInFlareNot$TimeNextVT[i] =="None"){
    MCInFlareNot$TempColFlare[i] = "after a flare"
  } else if (MCInFlareNot$TimePrevVT[i] =="None"){
    MCInFlareNot$TempColFlare[i] = "before a flare"
  } else if (MCInFlareNot$TimeNextVT[i] == 0){
    MCInFlareNot$TempColFlare[i] = "during a flare"
  } else if ((MCInFlareNot$TimeNextVT[i] != "None") & (MCInFlareNot$TimePrevVT[i] != "None")){ 
    if (as.numeric(MCInFlareNot$TimeNextVT[i]) + as.numeric(MCInFlareNot$TimePrevVT[i]) > 0){
      MCInFlareNot$TempColFlare[i] = "before a flare"
    } else {
      MCInFlareNot$TempColFlare[i] = "after a flare"
    }
  }
}


MCInFlareNot = MCInFlareNot[MCInFlareNot$TempColFlare!= "after a flare",]

MCInFlareNot = MCInFlareNot[,c(1, 8, 2, 3, 5, 9:645)]
write.table(MCInFlareNot, "InFlareNot.tsv", sep = "\t", quote = F, row.names = F)

## MaAsLin analysis 2: between patients >1 year quiescent disease versus in a flare 
Maaslin('InFlareNot.tsv','nOud Final MetaCyc Analysis 2',strInputConfig = '2.MetaCyc.read.config', dMinSamp = 0.25, fZeroInflated = T,strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'))




###### Analysis 3: categorical comparison of gut metagenome, all patients during a flare with
###### all patients before a flare

MCInFlareNot = MetaCycVT
## Remove patients that have NA in both Prev/Next <1 yr flare
MCInFlareNot<-MCInFlareNot[!with(MCInFlareNot,is.na(MCInFlareNot$TimeNextVT)& is.na(MCInFlareNot$TimePrevVT)),]

## Creating new column 'in flare/ not in flare >1 yr'
MCInFlareNot = cbind(MCInFlareNot[,1:7], "TempColFlare"=NA, MCInFlareNot[,8:ncol(MCInFlareNot)])

## Giving value to new column 'in flare/ not in flare >1 yr'
for (i in 1:nrow(MCInFlareNot)){
  if (is.na(MCInFlareNot$TimeNextVT[i])) {
    MCInFlareNot$TempColFlare[i]= "None"
    MCInFlareNot$TimeNextVT[i] = "None"
  } else if (is.na(MCInFlareNot$TimePrevVT[i])){
    MCInFlareNot$TimePrevVT[i] = "None"
  } else if (MCInFlareNot$TimeNextVT[i]< 0) {
    MCInFlareNot$TempColFlare[i] = "before a flare"
  } else {
    MCInFlareNot$TempColFlare[i] = "during a flare"
  }
}
#
for (i in 1:nrow(MCInFlareNot)){
  if (MCInFlareNot$TimeNextVT[i] =="None"){
    MCInFlareNot$TempColFlare[i] = "after a flare"
  } else if (MCInFlareNot$TimePrevVT[i] =="None"){
    MCInFlareNot$TempColFlare[i] = "before a flare"
  } else if (MCInFlareNot$TimeNextVT[i] == 0){
    MCInFlareNot$TempColFlare[i] = "during a flare"
  } else if ((MCInFlareNot$TimeNextVT[i] != "None") & (MCInFlareNot$TimePrevVT[i] != "None")){ 
    if (as.numeric(MCInFlareNot$TimeNextVT[i]) + as.numeric(MCInFlareNot$TimePrevVT[i]) > 0){
      MCInFlareNot$TempColFlare[i] = "before a flare"
    } else {
      MCInFlareNot$TempColFlare[i] = "after a flare"
    }
  }
}


MCInFlareNot = MCInFlareNot[MCInFlareNot$TempColFlare!= "before a flare",]

MCInFlareNot = MCInFlareNot[,c(1, 8, 2, 3, 5, 9:645)]
write.table(MCInFlareNot, "InFlareNot.tsv", sep = "\t", quote = F, row.names = F)

## MaAsLin analysis 3 
Maaslin('InFlareNot.tsv','nOud Final MetaCyc Analysis 3',strInputConfig = '3.MetaCyc.read.config', dMinSamp = 0.25, fZeroInflated = T,strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'))








############## Linear analysis 2a
### (Patients who have next flare < 1 year) - (time until next flare). 
MCCDIIa = MetaCycVT

MCCDIIa<-MCCDIIa[!with(MCCDIIa,is.na(MCCDIIa$TimeNextVT)& is.na(MCCDIIa$TimePrevVT)),]
#
MCCDIIa = cbind(MCCDIIa[,1:7], "LinBefore"=NA, MCCDIIa[,8:ncol(MCCDIIa)])
MCCDIIa$LinBefore = as.numeric(as.character(MCCDIIa$LinBefore))


for (i in 1:nrow(MCCDIIa)){
  if (is.na(MCCDIIa$TimeNextVT[i])) {
    MCCDIIa$LinBefore[i]= "None"
    MCCDIIa$TimeNextVT[i] = "None"
  } else if (is.na(MCCDIIa$TimePrevVT[i])){
    MCCDIIa$TimePrevVT[i] = "None"
  } else if (MCCDIIa$TimeNextVT[i]< 0) {
    MCCDIIa$LinBefore[i] = "before a flare"
  } else {
    MCCDIIa$LinBefore[i] = "during a flare"
  }
}

for (i in 1:nrow(MCCDIIa)){
  if (MCCDIIa$TimeNextVT[i] =="None"){
    MCCDIIa$LinBefore[i] = "after a flare"
  } else if (MCCDIIa$TimePrevVT[i] =="None"){
    MCCDIIa$LinBefore[i] = "before a flare"
  } else if (MCCDIIa$TimeNextVT[i] == 0){
    MCCDIIa$LinBefore[i] = "during a flare"
  } else if ((MCCDIIa$TimeNextVT[i] != "None") & (MCCDIIa$TimePrevVT[i] != "None")){ 
    if (as.numeric(MCCDIIa$TimeNextVT[i]) + as.numeric(MCCDIIa$TimePrevVT[i]) > 0){
      MCCDIIa$LinBefore[i] = "before a flare"
    } else {
      MCCDIIa$LinBefore[i] = "after a flare"
    }
  }
}

MCCDIIa = MCCDIIa[MCCDIIa$LinBefore!= "after a flare",]
MCCDIIa = MCCDIIa[MCCDIIa$LinBefore!= "during a flare",]
MCCDIIa$TimeNextVT = as.numeric(as.character(MCCDIIa$TimeNextVT))

MCCDIIa$TimeNextVT[MCCDIIa$TimeNextVT< (-182.5)] <- NA
MCCDIIa = MCCDIIa[!is.na(MCCDIIa$TimeNextVT),]
#

MCCDIIa = MCCDIIa[,c(1, 7, 2, 3, 5, 9:645)]

write.table(MCCDIIa, "LinBeforein1Yr.tsv", sep = "\t", quote = F, row.names = F)

### MaAsLin run 2a (Patients who have next flare < 6 mo) - (time until next flare). 
Maaslin('LinBeforein1Yr.tsv','nOud Final MetaCyc analyses 4a',strInputConfig = '2a.MetaCyc.read.config', dMinSamp = 0.25, fZeroInflated = T, strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'))







##### Analyses 2b: (Patients who had last flare < 1 year) - (time since last flare). 
MCLinAfterIIb = MetaCycVT

#
MCLinAfterIIb<-MCLinAfterIIb[!with(MCLinAfterIIb,is.na(MCLinAfterIIb$TimeNextVT)& is.na(MCLinAfterIIb$TimePrevVT)),]
#
MCLinAfterIIb = cbind(MCLinAfterIIb[,1:7], "LinAfter"=NA, MCLinAfterIIb[,8:ncol(MCLinAfterIIb)])
MCLinAfterIIb$LinAfter = as.numeric(as.character(MCLinAfterIIb$LinAfter))


for (i in 1:nrow(MCLinAfterIIb)){
  if (is.na(MCLinAfterIIb$TimeNextVT[i])) {
    MCLinAfterIIb$LinAfter[i]= "None"
    MCLinAfterIIb$TimeNextVT[i] = "None"
  } else if (is.na(MCLinAfterIIb$TimePrevVT[i])){
    MCLinAfterIIb$TimePrevVT[i] = "None"
  } else if (MCLinAfterIIb$TimeNextVT[i]< 0) {
    MCLinAfterIIb$LinAfter[i] = "before a flare"
  } else {
    MCLinAfterIIb$LinAfter[i] = "during a flare"
  }
}

for (i in 1:nrow(MCLinAfterIIb)){
  if (MCLinAfterIIb$TimeNextVT[i] =="None"){
    MCLinAfterIIb$LinAfter[i] = "after a flare"
  } else if (MCLinAfterIIb$TimePrevVT[i] =="None"){
    MCLinAfterIIb$LinAfter[i] = "before a flare"
  } else if (MCLinAfterIIb$TimeNextVT[i] == 0){
    MCLinAfterIIb$LinAfter[i] = "during a flare"
  } else if ((MCLinAfterIIb$TimeNextVT[i] != "None") & (MCLinAfterIIb$TimePrevVT[i] != "None")){ 
    if (as.numeric(MCLinAfterIIb$TimeNextVT[i]) + as.numeric(MCLinAfterIIb$TimePrevVT[i]) > 0){
      MCLinAfterIIb$LinAfter[i] = "before a flare"
    } else {
      MCLinAfterIIb$LinAfter[i] = "after a flare"
    }
  }
}

MCLinAfterIIb = MCLinAfterIIb[MCLinAfterIIb$LinAfter!= "before a flare",]
MCLinAfterIIb = MCLinAfterIIb[MCLinAfterIIb$LinAfter!= "during a flare",]
MCLinAfterIIb$TimePrevVT = as.numeric(as.character(MCLinAfterIIb$TimePrevVT))

#
MCLinAfterIIb$TimePrevVT[MCLinAfterIIb$TimePrevVT> 182.5]<-NA
MCLinAfterIIb = MCLinAfterIIb[!is.na(MCLinAfterIIb$TimePrevVT),]


MCLinAfterIIb = MCLinAfterIIb[,c(1, 6, 2, 3, 5, 9:645)]
write.table(MCLinAfterIIb, "LinAfterin1Yr.tsv", sep = "\t", quote = F, row.names = F)
##2b 
Maaslin('LinAfterin1Yr.tsv','nOud Final Metacyc Analysis 4b',strInputConfig = '2b.MetaCyc.read.config', dMinSamp = 0.25, fZeroInflated = T, strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'))

























##--------------------------------------------------------------------------
######## Virulence factors
## Setting my working Directory.  
setwd("~/Documents/Pilot Project - Virtual Time Line/working directory")

## Importing Valerie's/Floris/ Phenotype file.
db = read.csv("VALFLO.csv", header = T, sep = ";")
db = as.data.frame(db)

## Importing my metadata file (time relative to a flare).
VT = read.csv("VIRTUALTIMELINERDEF.csv", header = T, sep = ";")
VT = as.data.frame(VT)

## Merging Valerie's file with my file, to filter the patients who have
## no metagenomic sequencing data available. 
FinalVT = merge (db, VT, by="UMCGNoFromZIC", all = FALSE)
FinalVT=as.data.frame(FinalVT)
FinalVT = FinalVT[FinalVT$IncludedSamples == 'yes',]
FinalVT = FinalVT[,c("Sex", "UMCGIBDDNAID", "PFReads", "AgeAtFecalSampling", "TimeEndPreviousExacerbation", "TimeToStartNextExacerbation", "DiagnosisCurrent", "DiseaseLocation", "MedicationPPI", "AntibioticsWithin3MonthsPriorToSampling", "BMI")]
FinalVT = FinalVT[,c(2, 1, 3, 7, 4, 5, 6, 11, 8, 9, 10)]


## Importing Kraken metagenomic sequencing file 'Virulence Factors'. 
## This file is imported from Arnau. 
VirulenceFac = read.delim("virulence.txt", header = T, sep = "\t")
rownames(VirulenceFac) = paste(VirulenceFac$Gene, VirulenceFac$Product,VirulenceFac$VF_Name,VirulenceFac$Origin, sep = '_')
VirulenceFac = VirulenceFac[,5:ncol(VirulenceFac)]
VirulenceFac = as.data.frame(t(VirulenceFac))

VirulenceFac["UMCGIBDDNAID"] = row.names(VirulenceFac)
VirulenceFac = VirulenceFac[,c(1715, 1:1714)]

VirulenceFacVT = merge (FinalVT, VirulenceFac, by = "UMCGIBDDNAID", all = FALSE)


## Converting all negative numbers (patient is in a flare multiple days) into 
## zero's (meaning that all patients in a flare are stated as just 'in a flare' =
## 0 days until last flare and 0 days until next flare)
VirulenceFacVT = cbind(VirulenceFacVT[,1:6], "TimePrevVT"=NA, "TimeToStartNextExacerbation"=VirulenceFacVT$TimeToStartNextExacerbation, "TimeNextNegtoZer"=NA, VirulenceFacVT[,8:ncol(VirulenceFacVT)])

VirulenceFacVT$TimeEndPreviousExacerbation = as.numeric(as.character(VirulenceFacVT$TimeEndPreviousExacerbation))
VirulenceFacVT$TimeToStartNextExacerbation = as.numeric(as.character(VirulenceFacVT$TimeToStartNextExacerbation))

for (i in 1:nrow(VirulenceFacVT)) {
  if (VirulenceFacVT$TimeEndPreviousExacerbation[i] < 0 & !is.na(VirulenceFacVT$TimeEndPreviousExacerbation[i])) {
    VirulenceFacVT$TimePrevVT[i] = 0
    VirulenceFacVT$TimeNextNegtoZer[i] = 0
  } else {
    VirulenceFacVT$TimePrevVT[i] = VirulenceFacVT$TimeEndPreviousExacerbation[i]
    VirulenceFacVT$TimeNextNegtoZer[i] = VirulenceFacVT$TimeToStartNextExacerbation[i]
  }
}

## Making 'days to the next flare' all zero. 
VirulenceFacVT = cbind(VirulenceFacVT[,1:9], "TimeNextVT"=NA, VirulenceFacVT[,10:ncol(VirulenceFacVT)])
for (i in 1:nrow(VirulenceFacVT)) {
  if (VirulenceFacVT$TimeNextNegtoZer[i] > 0 & !is.na(VirulenceFacVT$TimeNextNegtoZer[i])) {
    VirulenceFacVT$TimeNextVT[i] = ((VirulenceFacVT$TimeNextNegtoZer[i])*-1)
  }
  else {
    VirulenceFacVT$TimeNextVT[i] = VirulenceFacVT$TimeNextNegtoZer[i]
  }
}


VirFacCD = VirulenceFacVT[VirulenceFacVT$DiagnosisCurrent == 'CD',]
VirFacCD = VirFacCD[,c(1:5, 7, 10, 11:1728)]

# When PPI use is not documented in patient, it is agreed that we report 'no PPI use'.
for (i in 1:nrow(VirFacCD)){
  if (is.na(VirFacCD$MedicationPPI[i])){
    VirFacCD$MedicationPPI[i] = "no"
  } else 
    VirFacCD$MedicationPPI[i] = VirFacCD$MedicationPPI[i]
}

# When Antibiotic use is not documented in patient, it is agreed that we report 'no ab use'. 
for (i in 1:nrow(VirFacCD)){
  if (is.na(VirFacCD$AntibioticsWithin3MonthsPriorToSampling[i])){
    VirFacCD$AntibioticsWithin3MonthsPriorToSampling[i] = "no"
  } else 
    VirFacCD$AntibioticsWithin3MonthsPriorToSampling[i] = VirFacCD$AntibioticsWithin3MonthsPriorToSampling[i]
}

# Zorgen dat het helemaal klopt, dan alle NextVT die 0 zijn, dan ook PrevVT hebben die nul is.
for (i in 1:nrow(VirFacCD)){
  if (!is.na(VirFacCD$TimePrevVT[i]) & VirFacCD$TimePrevVT[i] == 0 ){
    VirFacCD$TimeNextVT[i] = 0
  } else 
    VirFacCD$TimeNextVT[i] = VirFacCD$TimeNextVT[i]
}

for (i in 1:nrow(VirFacCD)){
  if (!is.na(VirFacCD$TimeNextVT[i]) & VirFacCD$TimeNextVT[i] == 0 ){
    VirFacCD$TimePrevVT[i] = 0
  } else 
    VirFacCD$TimePrevVT[i] = VirFacCD$TimePrevVT[i]
}

### Filtering out species that are abundant in <5% of patients
VF_Filter = VirFacCD[,c(1, 12: 1725)]
VF_Filter2 = VF_Filter[,-1]
rownames(VF_Filter2) = VF_Filter[,1]
VF_Filter2 = as.data.frame(t(VF_Filter2))

VF_Filter2 = VF_Filter2[rowSums(VF_Filter2==0.00)<=ncol(VF_Filter2)*0.95,]
VF_Filter2 = as.data.frame(t(VF_Filter2))
VF_Filter2["UMCGIBDDNAID"] = row.names(VF_Filter2)
VF_Filter2=VF_Filter2[,c(888, 1:887)]


VF_VT = VirFacCD[,c(1:11)]
VF_VT = merge(VF_VT, VF_Filter2, by= "UMCGIBDDNAID", all = FALSE)



## Er zijn spaties en slashes in de kolomnamen. Hier kan MaAslin niet mee werken
## Nu wil ik dus alle ':' vervangen door '_'.
names(VF_VT) = gsub(x = names(VF_VT), pattern = " ", replacement = "_") 
names(VF_VT) = gsub(x = names(VF_VT), pattern = "/", replacement = "_") 
names(VF_VT) = gsub(x = names(VF_VT), pattern = ")", replacement = "_")
VFTidy = make.names(colnames(VF_VT), unique = TRUE)
colnames(VF_VT) = VFTidy





###### Analysis 1: categorical comparison of gut metagenome, all patients in a flare with all patients not in a flare

VFInFlareNot = VF_VT
## Remove patients that have NA in both Prev/Next <1 yr flare
VFInFlareNot<-VFInFlareNot[!with(VFInFlareNot,is.na(VFInFlareNot$TimeNextVT)& is.na(VFInFlareNot$TimePrevVT)),]

VFInFlareNot = cbind(VFInFlareNot[,1:7], "InFlareNot"=NA, VFInFlareNot[,8:ncol(VFInFlareNot)])
VFInFlareNot$InFlareNot = as.numeric(as.character(VFInFlareNot$InFlareNot))

for (i in 1:nrow(VFInFlareNot)){
  if (is.na(VFInFlareNot$TimeNextVT[i]) | is.na(VFInFlareNot$TimePrevVT[i])){
    VFInFlareNot$InFlareNot[i] = "Not in a flare"
  } else if (VFInFlareNot$TimeNextVT[i]== 0 | VFInFlareNot$TimePrevVT==0){
    VFInFlareNot$InFlareNot[i] = "In a flare"
  } else if (VFInFlareNot$TimeNextVT[i] <0 & VFInFlareNot$TimePrevVT >0) {
    VFInFlareNot$InFlareNot[i] = "Not in a flare"
  } else
    VFInFlareNot$InFlareNot[i] = "Not in a flare"
}


VFInFlareNot = VFInFlareNot[,c(1, 8, 2, 3, 5, 9:899)]
write.table(VFInFlareNot, "InFlareNot.tsv", sep = "\t", quote = F, row.names = F)


## MaAsLin analysis 1: between patients >1 year quiescent disease versus in a flare 
Maaslin('InFlareNot.tsv','nOud Final VF analysis 1',strInputConfig = '1.VirFac.read.config', dMinSamp = 0.25, fZeroInflated = T,strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'), strTransform = "none")







###### Analysis 2: categorical comparison of gut metagenome, all patients before flare with
###### all patients during a flare
VFInFlareNot = VF_VT

## Remove patients that have NA in both Prev/Next <1 yr flare
VFInFlareNot<-VFInFlareNot[!with(VFInFlareNot,is.na(VFInFlareNot$TimeNextVT)& is.na(VFInFlareNot$TimePrevVT)),]

## Creating new column 'in flare/ not in flare >1 yr'
VFInFlareNot = cbind(VFInFlareNot[,1:7], "TempColFlare"=NA, VFInFlareNot[,8:ncol(VFInFlareNot)])

## Giving value to new column 'in flare/ not in flare >1 yr'
for (i in 1:nrow(VFInFlareNot)){
  if (is.na(VFInFlareNot$TimeNextVT[i])) {
    VFInFlareNot$TempColFlare[i]= "None"
    VFInFlareNot$TimeNextVT[i] = "None"
  } else if (is.na(VFInFlareNot$TimePrevVT[i])){
    VFInFlareNot$TimePrevVT[i] = "None"
  } else if (VFInFlareNot$TimeNextVT[i]< 0) {
    VFInFlareNot$TempColFlare[i] = "before a flare"
  } else {
    VFInFlareNot$TempColFlare[i] = "during a flare"
  }
}
#
for (i in 1:nrow(VFInFlareNot)){
  if (VFInFlareNot$TimeNextVT[i] =="None"){
    VFInFlareNot$TempColFlare[i] = "after a flare"
  } else if (VFInFlareNot$TimePrevVT[i] =="None"){
    VFInFlareNot$TempColFlare[i] = "before a flare"
  } else if (VFInFlareNot$TimeNextVT[i] == 0){
    VFInFlareNot$TempColFlare[i] = "during a flare"
  } else if ((VFInFlareNot$TimeNextVT[i] != "None") & (VFInFlareNot$TimePrevVT[i] != "None")){ 
    if (as.numeric(VFInFlareNot$TimeNextVT[i]) + as.numeric(VFInFlareNot$TimePrevVT[i]) > 0){
      VFInFlareNot$TempColFlare[i] = "before a flare"
    } else {
      VFInFlareNot$TempColFlare[i] = "after a flare"
    }
  }
}

VFInFlareNot = VFInFlareNot[VFInFlareNot$TempColFlare!= "after a flare",]

VFInFlareNot = VFInFlareNot[,c(1, 8, 2, 3, 5, 9: 899)]
write.table(VFInFlareNot, "InFlareNot.tsv", sep = "\t", quote = F, row.names = F)

## MaAsLin analysis 2: between patients >1 year quiescent disease versus in a flare 
Maaslin('InFlareNot.tsv','nOud Final VF analysis 2',strInputConfig = '2.VirFac.read.config', dMinSamp = 0.25, fZeroInflated = T,strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'), strTransform = "none")






###### Analysis 3: categorical comparison of gut metagenome, all patients before flare with
###### all patients during a flare
VFInFlareNot = VF_VT

## Remove patients that have NA in both Prev/Next <1 yr flare
VFInFlareNot<-VFInFlareNot[!with(VFInFlareNot,is.na(VFInFlareNot$TimeNextVT)& is.na(VFInFlareNot$TimePrevVT)),]

## Creating new column 'in flare/ not in flare >1 yr'
VFInFlareNot = cbind(VFInFlareNot[,1:7], "TempColFlare"=NA, VFInFlareNot[,8:ncol(VFInFlareNot)])

## Giving value to new column 'in flare/ not in flare >1 yr'
for (i in 1:nrow(VFInFlareNot)){
  if (is.na(VFInFlareNot$TimeNextVT[i])) {
    VFInFlareNot$TempColFlare[i]= "None"
    VFInFlareNot$TimeNextVT[i] = "None"
  } else if (is.na(VFInFlareNot$TimePrevVT[i])){
    VFInFlareNot$TimePrevVT[i] = "None"
  } else if (VFInFlareNot$TimeNextVT[i]< 0) {
    VFInFlareNot$TempColFlare[i] = "before a flare"
  } else {
    VFInFlareNot$TempColFlare[i] = "during a flare"
  }
}
#
for (i in 1:nrow(VFInFlareNot)){
  if (VFInFlareNot$TimeNextVT[i] =="None"){
    VFInFlareNot$TempColFlare[i] = "after a flare"
  } else if (VFInFlareNot$TimePrevVT[i] =="None"){
    VFInFlareNot$TempColFlare[i] = "before a flare"
  } else if (VFInFlareNot$TimeNextVT[i] == 0){
    VFInFlareNot$TempColFlare[i] = "during a flare"
  } else if ((VFInFlareNot$TimeNextVT[i] != "None") & (VFInFlareNot$TimePrevVT[i] != "None")){ 
    if (as.numeric(VFInFlareNot$TimeNextVT[i]) + as.numeric(VFInFlareNot$TimePrevVT[i]) > 0){
      VFInFlareNot$TempColFlare[i] = "before a flare"
    } else {
      VFInFlareNot$TempColFlare[i] = "after a flare"
    }
  }
}

VFInFlareNot = VFInFlareNot[VFInFlareNot$TempColFlare!= "before a flare",]

VFInFlareNot = VFInFlareNot[,c(1, 8, 2, 3, 5, 9: 899)]
write.table(VFInFlareNot, "InFlareNot.tsv", sep = "\t", quote = F, row.names = F)

## MaAsLin analysis 3: between patients >1 year quiescent disease versus in a flare 
Maaslin('InFlareNot.tsv','nOud Final VF analysis 3',strInputConfig = '3.VirFac.read.config', dMinSamp = 0.25, fZeroInflated = T,strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'), strTransform = "none")












############## Linear analysis 2a
### (Patients who have next flare < 1 year) - (time until next flare). 
VF_CDIIa = VF_VT
#
VF_CDIIa<-VF_CDIIa[!with(VF_CDIIa,is.na(VF_CDIIa$TimeNextVT)& is.na(VF_CDIIa$TimePrevVT)),]
#
VF_CDIIa = cbind(VF_CDIIa[,1:7], "LinBefore"=NA, VF_CDIIa[,8:ncol(VF_CDIIa)])
VF_CDIIa$LinBefore = as.numeric(as.character(VF_CDIIa$LinBefore))


for (i in 1:nrow(VF_CDIIa)){
  if (is.na(VF_CDIIa$TimeNextVT[i])) {
    VF_CDIIa$LinBefore[i]= "None"
    VF_CDIIa$TimeNextVT[i] = "None"
  } else if (is.na(VF_CDIIa$TimePrevVT[i])){
    VF_CDIIa$TimePrevVT[i] = "None"
  } else if (VF_CDIIa$TimeNextVT[i]< 0) {
    VF_CDIIa$LinBefore[i] = "before a flare"
  } else {
    VF_CDIIa$LinBefore[i] = "during a flare"
  }
}

for (i in 1:nrow(VF_CDIIa)){
  if (VF_CDIIa$TimeNextVT[i] =="None"){
    VF_CDIIa$LinBefore[i] = "after a flare"
  } else if (VF_CDIIa$TimePrevVT[i] =="None"){
    VF_CDIIa$LinBefore[i] = "before a flare"
  } else if (VF_CDIIa$TimeNextVT[i] == 0){
    VF_CDIIa$LinBefore[i] = "during a flare"
  } else if ((VF_CDIIa$TimeNextVT[i] != "None") & (VF_CDIIa$TimePrevVT[i] != "None")){ 
    if (as.numeric(VF_CDIIa$TimeNextVT[i]) + as.numeric(VF_CDIIa$TimePrevVT[i]) > 0){
      VF_CDIIa$LinBefore[i] = "before a flare"
    } else {
     VF_CDIIa$LinBefore[i] = "after a flare"
    }
  }
}

VF_CDIIa = VF_CDIIa[VF_CDIIa$LinBefore!= "after a flare",]
VF_CDIIa = VF_CDIIa[VF_CDIIa$LinBefore!= "during a flare",]


VF_CDIIa$TimeNextVT = as.numeric(as.character(VF_CDIIa$TimeNextVT))
VF_CDIIa$TimeNextVT[VF_CDIIa$TimeNextVT< (-182.5)]<-NA
VF_CDIIa = VF_CDIIa[!is.na(VF_CDIIa$TimeNextVT),]


VF_CDIIa = VF_CDIIa[,c(1, 7, 2, 3, 5, 9:899)]

write.table(VF_CDIIa, "LinBeforein1Yr.tsv", sep = "\t", quote = F, row.names = F)
### MaAsLin run 2a (Patients who have next flare < 1 year) - (time until next flare). 
Maaslin('LinBeforein1Yr.tsv','nOud Vir Fac analyses 4a',strInputConfig = '2a.VirFac.read.config', dMinSamp = 0.25, fZeroInflated = T, strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'), strTransform = "none")







##### Analyses 2b: (Patients who had last flare < 1 year) - (time since last flare). 
VF_LinAfterIIb = VF_VT

#
VF_LinAfterIIb<-VF_LinAfterIIb[!with(VF_LinAfterIIb,is.na(VF_LinAfterIIb$TimeNextVT)& is.na(VF_LinAfterIIb$TimePrevVT)),]
#
VF_LinAfterIIb = cbind(VF_LinAfterIIb[,1:7], "LinAfter"=NA, VF_LinAfterIIb[,8:ncol(VF_LinAfterIIb)])
VF_LinAfterIIb$LinAfter = as.numeric(as.character(VF_LinAfterIIb$LinAfter))


for (i in 1:nrow(VF_LinAfterIIb)){
  if (is.na(VF_LinAfterIIb$TimeNextVT[i])) {
    VF_LinAfterIIb$LinAfter[i]= "None"
    VF_LinAfterIIb$TimeNextVT[i] = "None"
  } else if (is.na(VF_LinAfterIIb$TimePrevVT[i])){
    VF_LinAfterIIb$TimePrevVT[i] = "None"
  } else if (VF_LinAfterIIb$TimeNextVT[i]< 0) {
    VF_LinAfterIIb$LinAfter[i] = "before a flare"
  } else {
    VF_LinAfterIIb$LinAfter[i] = "during a flare"
  }
}

for (i in 1:nrow(VF_LinAfterIIb)){
  if (VF_LinAfterIIb$TimeNextVT[i] =="None"){
    VF_LinAfterIIb$LinAfter[i] = "after a flare"
  } else if (VF_LinAfterIIb$TimePrevVT[i] =="None"){
    VF_LinAfterIIb$LinAfter[i] = "before a flare"
  } else if (VF_LinAfterIIb$TimeNextVT[i] == 0){
    VF_LinAfterIIb$LinAfter[i] = "during a flare"
  } else if ((VF_LinAfterIIb$TimeNextVT[i] != "None") & (VF_LinAfterIIb$TimePrevVT[i] != "None")){ 
    if (as.numeric(VF_LinAfterIIb$TimeNextVT[i]) + as.numeric(VF_LinAfterIIb$TimePrevVT[i]) > 0){
     VF_LinAfterIIb$LinAfter[i] = "before a flare"
    } else {
      VF_LinAfterIIb$LinAfter[i] = "after a flare"
    }
  }
}
#
VF_LinAfterIIb = VF_LinAfterIIb[VF_LinAfterIIb$LinAfter!= "before a flare",]
VF_LinAfterIIb = VF_LinAfterIIb[VF_LinAfterIIb$LinAfter!= "during a flare",]


VF_LinAfterIIb$TimePrevVT = as.numeric(as.character(VF_LinAfterIIb$TimePrevVT))
VF_LinAfterIIb$TimePrevVT[VF_LinAfterIIb$TimePrevVT > (182.5)]<-NA
VF_LinAfterIIb = VF_LinAfterIIb[!is.na(VF_LinAfterIIb$TimePrevVT),]
#


VF_LinAfterIIb = VF_LinAfterIIb[,c(1, 6, 2, 3, 5, 9:899)]
write.table(VF_LinAfterIIb, "LinAfterin1Yr.tsv", sep = "\t", quote = F, row.names = F)
##2b 
Maaslin('LinAfterin1Yr.tsv','nOud Virulence fac analyses 4b',strInputConfig = '2b.VirFac.read.config', dMinSamp = 0.25, fZeroInflated = T, strForcedPredictors = c('Sex', 'PFReads', 'AgeAtFecalSampling', 'BMI', 'DiseaseLocation', 'MedicationPPI', 'AntibioticsWithin3MonthsPriorToSampling'), strTransform = "none")














