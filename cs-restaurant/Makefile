#
# Experiment management for Tgen
#

# Environment settings
SHELL = /bin/bash
TGEN = ../run_tgen.py
MEM = 16g
CPUS = 8
QP = --queue=cpu-troja.q
ifdef LOCAL
  QP = --engine=console
endif
QSUBMIT = qsubmit --logdir '$(TRY_DIR)' --mem $(MEM) --cores $(CPUS) --jobname T.$(TRY_NUM)$(SEED).$(RUN_NAME) $(QP) $(QHOLD) --
ACTIVATE = echo -n

ifdef DEBUG
  DEBUG_LOG_TRAIN = -d $(TRY_DIR)/train.debug-log.txt.gz
  DEBUG_LOG_GEN = -d $(TRY_DIR)/gen.debug-log.txt
endif
ifdef DEBUG_GEN
  DEBUG_LOG_GEN = -d $(TRY_DIR)/gen.debug-log.txt
endif
METRICS = $$HOME/e2e-metrics/measure_scores.py
POSTPROCESS = ./postprocess/postprocess.py
AVERAGE_SCORES = ../util/average_scores.pl --e2e
GREP_SCORES = ../util/grep_scores.pl --bleu-range=8:20 --nist-range=2.5:4.5 --cider-range=0.5:2 --meteor-range=12:25 --rouge-range=25:50
DESC_EXP = ../util/describe_experiment.py
CFG_OVERRIDE_TOOL = ../util/cfg_override.py
NOWRAP = ../util/nowrap.pl
NUM_EXP = 20  # number of experiments to list in `make desc'

LOGFILES = $$file/T.* $$file/{asearch_gen.,gen.,seq2seq_gen.,}log.txt # logfile pattern to use for score printing
SCOREFILE = SCORE

# Runs directories
RUNS_DIR  := runs# main directory for experiment outputs
TRY_NUM   := $(shell perl -e '$$m=0; for(<$(RUNS_DIR)/*>){/\/(\d+)_/ and $$1 > $$m and $$m=$$1;} printf "%03d", $$m+1;')# experiment number
RUN_NAME  := experiment# default name, to be overridden by targets
DATE      := $(shell date +%Y-%m-%d_%H-%M-%S)
TRY_DIR    = $(RUNS_DIR)/$(TRY_NUM)_$(DATE)_$(RUN_NAME)# experiment output directory
TRAINING_SET := training2-delex

ifdef D # Shortcut D -> DESC
  DESC := $(D)
endif
ifdef J # Shortcut J -> JOBS
  JOBS := $(J)
endif
ifdef R # Shortcut R -> REFRESH
  REFRESH := $(R)
endif
ifdef N # Shortcut R -> REFRESH
  NUM_EXP := $(N)
endif

ifdef JOBS
  PARALLEL := -j $(JOBS) -e "T.$(TRY_NUM)"
endif

# Input data file defaults

CONFIG = config/config.yaml
TRAIN_PORTION = 1.0

TRAIN_DAS = data/$(TRAINING_SET)/train-das.txt
TRAIN_TREES = data/$(TRAINING_SET)/train-text.pickle.gz
TRAIN_TOKS = data/$(TRAINING_SET)/train-text.conll
TRAIN_ABSTR = data/$(TRAINING_SET)/train-abst.txt
TRAIN_CONC = data/$(TRAINING_SET)/train-text_l.txt

GRAMMATEMES = data/$(TRAINING_SET)/grammatemes.tsv
SURFACE_FORMS = data/$(TRAINING_SET)/surface_forms.json

TEST_DAS = data/$(TRAINING_SET)/devel-das.txt
TEST_TREES = data/$(TRAINING_SET)/devel-text.pickle.gz
TEST_TOKS = data/$(TRAINING_SET)/devel-text.conll
TEST_ABSTR = data/$(TRAINING_SET)/devel-abst.txt
TEST_CONC = data/$(TRAINING_SET)/devel-text_l.txt
ifdef EVAL_TEST
  TEST_DAS = data/$(TRAINING_SET)/test-das.txt
  TEST_TREES = data/$(TRAINING_SET)/test-text.pickle.gz
  TEST_TOKS = data/$(TRAINING_SET)/test-text.conll
  TEST_ABSTR = data/$(TRAINING_SET)/test-abst.txt
  TEST_CONC = data/$(TRAINING_SET)/test-text_l.txt

  VALID_ABSTR = data/$(TRAINING_SET)/devel-abst.txt
  VALID_DAS = data/$(TRAINING_SET)/devel-das.txt
  VALID_TREES = data/$(TRAINING_SET)/devel-text.pickle.gz
  VALID_TOKS = data/$(TRAINING_SET)/devel-text.conll
endif

EVAL_SELECTOR =
TARGET_SELECTOR = gen

# Help text

define HELP_MESSAGE
Tgen experiments
================

make desc [NUM_EXP=20] [REFRESH=1]
* List experiments with descriptions and (possibly) scores.
* Change NUM_EXP if you need to see more than 20 latest results.
* Refresh scores to obtain new results by setting REFRESH=1 (makes it slower).

make seq2seq_cv_run J=[XX] D='Description'
* Run an experiment where seq2seq
  training is directly followed by testing generation.
  - set RANDS=1 to use 5 different random seeds (and average the results)
  - set EVAL_TEST=1 to evaluate on test data instead of development data

make score-XXX
* Retrieve a score for run without RANDS.

make cv_score-XXX
* Retrieve a score for run with RANDS.
---

Use QP='--console' if you want the job to run interactively instead
of submitting to the cluster.

Use ACTIVATE='source path/to/virtualenv/bin/activate' to run
within a Virtualenv instead of the default Python.

Use PYPY=1 to activate the preset PyPy Virtualenv.

Use DEBUG=1 to activate debugging outputs. Use DEBUG_GEN=1 to activate
debugging only for testing (in CV runs).
endef
export HELP_MESSAGE

#
# Targets
#

# Auxiliary targets

help:
	@echo "$$HELP_MESSAGE" | egrep --color '^(\s*make.*|)'

# List all experiments, find scores in logs (of various versions)
desc:
	@COLS=`tput cols`; \
	ls -d $(RUNS_DIR)/* | sort | tail -n $(NUM_EXP) | while read file; do \
		if [[ -f $$file/$(SCOREFILE) && -z "$(REFRESH)" ]]; then \
			cat $$file/$(SCOREFILE) | $(NOWRAP) $$COLS; \
			continue ; \
		fi ; \
		echo -ne $$file ":  " | sed 's/runs\///;s/_/  /;s/_/ /;s/_/  /;s/  20/  /;s/-[0-9][0-9]  /  /;' > $$file/$(SCOREFILE);  \
		$(GREP_SCORES) $(LOGFILES) >> $$file/$(SCOREFILE) ; \
		echo -ne '    ' >> $$file/$(SCOREFILE) ; \
		cat $$file/ABOUT | tr  '\n' ' ' >> $$file/$(SCOREFILE); \
		echo >> $$file/$(SCOREFILE); \
		cat $$file/$(SCOREFILE) | $(NOWRAP) $$COLS; \
	done

shortdesc:
	make desc NUM_EXP=$(NUM_EXP) | sed 's/\s\+.*F ..\...\s*/ /;s/\s\+.*cv_run :\s*/ /;s/E50.*+sort//;s/+cf_rnn.*128//;'

printvars:
	$(foreach V, $(sort $(.VARIABLES)), $(if $(filter-out environment% default automatic, $(origin $V)), $(info $V=$($V) ($(value $V)))))

printgit:
	@git status
	@echo -e "\n*** *** ***\n"
	@git log --pretty=format:"%h - %an, %ar : %s" -1
	@echo -e "\n*** *** ***\n"
	@git diff


prepare_dir:
	# create the experiment directory, print description and git version
	mkdir -p $(TRY_DIR)
	cp $(CONFIG) $(TRY_DIR)/config.yaml
	if [[ -n '$(CFG_OVERRIDE)' ]]; then \
		$(CFG_OVERRIDE_TOOL) $(TRY_DIR)/config.yaml '$(CFG_OVERRIDE)' ; \
	fi;
	[[ -n "$(DEBUG)" || -n "$(DEBUG_GEN)" ]] && DEBUG_SETTING=-d ; \
	[[ -n "$(RANDS)" ]] && RANDS_SETTING="-r $(RANDS)"; \
	[[ -n "$(EVAL_TEST)" ]] && EVAL_SETTING=-e ; \
	$(DESC_EXP) -t "$(TRAINING_SET)" -p "$(TRAIN_PORTION)" $$EVAL_SETTING $$DEBUG_SETTING $$RANDS_SETTING $(TRY_DIR)/config.yaml > $(TRY_DIR)/ABOUT ; \
	echo "$(DESC)" >> $(TRY_DIR)/ABOUT
	make printvars > $(TRY_DIR)/VARS
	make printgit > $(TRY_DIR)/GIT_VERSION

# Main targets

seq2seq_cv_run: cv_run

cv_run: RUN_NAME := cv_run
cv_run: prepare_dir
cv_run:
	if [[ -n "$(RANDS)" ]]; then \
		if [[ "$(RANDS)" = 1 ]]; then \
			SEEDS=(s0 s1 s2 s3 s4); \
		else \
			SEEDS=(`seq -f "s%1.0f" 0 $$(($(RANDS) - 1))`); \
		fi; \
	else \
		SEEDS=(""); \
	fi; \
	for SEED in "$${SEEDS[@]}"; do \
		TRAIN_RUN=seq2seq_train ; \
		TEST_RUN=seq2seq_gen_cv ; \
		make $$TRAIN_RUN TRY_DIR=$(TRY_DIR)/$$SEED SEED=$$SEED TRY_NUM=$(TRY_NUM) CFG_OVERRIDE='$(CFG_OVERRIDE)' 2>&1 | tee -a $(TRY_DIR)/cv.log.txt ; \
		JOB_NUM=`cat $(TRY_DIR)/cv.log.txt | grep '^Your job' | tail -n 1 | sed 's/^Your job \([0-9]\+\).*/\1/;' ` ; \
		make $$TEST_RUN TRY_DIR=$(TRY_DIR)/$$SEED SEED=$$SEED TRY_NUM=$(TRY_NUM) QP="$(QP)" QHOLD="--hold $$JOB_NUM"; \
	done

cv_score-%:
	$(eval TRY_DIR := $(shell ls -d $(RUNS_DIR)/$**))
	$(eval TRY_NUM := $*)
	make cv_score TRY_DIR=$(TRY_DIR) TRY_NUM=$(TRY_NUM)

cv_score:
	$(AVERAGE_SCORES) $(TRY_DIR)/*/T.*.*_gen.o* | tee $(TRY_DIR)/gen.log.txt

score-%:
	$(eval TRY_DIR := $(shell ls -d $(RUNS_DIR)/$**))
	$(eval TRY_NUM := $*)
	make score TRY_DIR=$(TRY_DIR) TRY_NUM=$(TRY_NUM)

score:
	$(AVERAGE_SCORES) $(TRY_DIR)/T.*.*_gen.o* | tee $(TRY_DIR)/gen.log.txt

seq2seq_train: RUN_NAME := seq2seq_train
seq2seq_train: prepare_dir
seq2seq_train:
	# copy needed files (to ensure replication), then run the experiment
	cp $(TRAIN_ABSTR) $(TRY_DIR)/train-abst.txt
	cp $(TRAIN_DAS) $(TRY_DIR)/train-das.txt
	cp $(SURFACE_FORMS) $(TRY_DIR)/surface_forms.json
	if [[ -n "$(TOKS)" ]]; then \
		cp $(TRAIN_TOKS) $(TRY_DIR)/train-text.conll; \
		TRAIN_DATA=$(TRY_DIR)/train-text.conll ; \
	else \
		cp $(TRAIN_TREES) $(TRY_DIR)/train-text.pickle.gz; \
		TRAIN_DATA=$(TRY_DIR)/train-text.pickle.gz ; \
	fi; \
	if [[ -n "$(EVAL_TEST)" ]]; then \
		cp $(VALID_ABSTR) $(TRY_DIR)/devel-abst.txt ; \
		cp $(VALID_DAS) $(TRY_DIR)/devel-das.txt ; \
		if [[ -n "$(TOKS)" ]]; then \
		    cp $(VALID_TOKS) $(TRY_DIR)/devel-text.conll ; \
		    VALID_DATA_SPEC="-v \"$(TRY_DIR)/devel-das.txt,$(TRY_DIR)/devel-text.conll\""; \
		else \
		    cp $(VALID_TREES) $(TRY_DIR)/devel-text.pickle.gz ; \
		    VALID_DATA_SPEC="-v \"$(TRY_DIR)/devel-das.txt,$(TRY_DIR)/devel-text.pickle.gz\""; \
		fi ; \
		LEXIC_SPEC="$(TRY_DIR)/surface_forms.json,$(TRY_DIR)/train-abst.txt,$(TRY_DIR)/devel-abst.txt"; \
	else \
		LEXIC_SPEC="$(TRY_DIR)/surface_forms.json,$(TRY_DIR)/train-abst.txt"; \
	fi; \
	$(QSUBMIT) '$(ACTIVATE); \
		$(TGEN) seq2seq_train $(DEBUG_LOG_TRAIN) \
		-s $(TRAIN_PORTION) $(PARALLEL) -r "$(SEED)" '"$$VALID_DATA_SPEC"' \
		-l '"$$LEXIC_SPEC"' \
		$(TRY_DIR)/config.yaml $(TRY_DIR)/train-das.txt '$$TRAIN_DATA' \
		$(TRY_DIR)/seq2seq.pickle.gz' 2>&1 | tee $(TRY_DIR)/seq2seq_train.log.txt

seq2seq_gen_cv:
	make prepare_test_data TRY_DIR=$(TRY_DIR)
	make seq2seq_gen_process TRY_DIR=$(TRY_DIR) TRY_NUM=$(TRY_NUM) QP="$(QP)" QHOLD="$(QHOLD)"

prepare_test_data:
	cp $(TEST_DAS) $(TRY_DIR)/test-das.txt
	cp $(TEST_ABSTR) $(TRY_DIR)/test-abst.txt
	cp $(TEST_CONC) $(TRY_DIR)/test-conc.txt
	if [[ -n "$(TOKS)" ]]; then \
		cp $(TEST_TOKS) $(TRY_DIR)/test-text.conll ; \
	else \
		cp $(TEST_TREES) $(TRY_DIR)/test-text.pickle.gz ; \
	        cp $(GRAMMATEMES) $(TRY_DIR)/grammatemes.tsv ; \
	fi


seq2seq_gen_process: RUN_NAME := seq2seq_gen
seq2seq_gen_process: GEN_MODEL := $(TRY_DIR)/seq2seq.pickle.gz
seq2seq_gen_process: SELECTOR_CONFIG := -r "$(EVAL_SELECTOR)" -t "$(TARGET_SELECTOR)"
seq2seq_gen_process: gen_process

gen_process:
	# run
	if [[ -n "$(TOKS)" ]]; then \
		TEST_DATA=$(TRY_DIR)/test-conc.txt ; \
	else \
		TEST_DATA=$(TRY_DIR)/test-text.pickle.gz ; \
	fi; \
	$(QSUBMIT) '$(ACTIVATE); \
		$(TGEN) $(RUN_NAME) -e '$$TEST_DATA' $(DEBUG_LOG_GEN) \
		$(SELECTOR_CONFIG) \
		-D $(TRY_DIR)/out-delex.txt \
		-w $(TRY_DIR)/out-trees.yaml.gz -a $(TRY_DIR)/test-abst.txt \
		$(GEN_MODEL) $(TRY_DIR)/test-das.txt; \
		$(POSTPROCESS) -m $(GEN_MODEL) -i $(TRY_DIR)/test-das.txt -f $(TRY_DIR)/surface_forms.json \
			-g $(TRY_DIR)/grammatemes.tsv -s $(TARGET_SELECTOR) \
			$(TRY_DIR)/out-trees.yaml.gz $(TRY_DIR)/out-text.txt; \
		$(METRICS) $(TRY_DIR)/test-conc.txt $(TRY_DIR)/out-text.txt' \
		2>&1 | tee $(TRY_DIR)/$(RUN_NAME).log.txt

# Rerun an experiment (TRY_DIR, TRY_NUM, RUN_NAME must be set!)
rerun:
	LAST_LOGFILE=`ls -t $(TRY_DIR)/T.*.$(RUN_NAME).* | head -n 1` ; \
	COMMAND=`cat $$LAST_LOGFILE | sed '1,/^== Directory/d;/^== Hard res/,$$d;s/^== Command:\s*//;'` ; \
	echo "Rerunning command: $$COMMAND"; \
	$(QSUBMIT) "$$COMMAND" 2>&1 | tee -a $(TRY_DIR)/$(RUN_NAME).log.txt ;

