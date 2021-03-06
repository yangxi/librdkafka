TESTSRCS_C = $(wildcard [08]*-*.c)
TESTSRCS_CXX= $(wildcard [08]*-*.cpp)
OBJS	  = $(TESTSRCS_C:%.c=%.o) $(TESTSRCS_CXX:%.cpp=%.o)

BIN	  = test-runner
LIBS	 += -lrdkafka++ -lrdkafka -lstdc++
OBJS	 += test.o testcpp.o tinycthread.o tinycthread_extra.o rdlist.o sockem.o sockem_ctrl.o
CFLAGS += -I../src
CXXFLAGS += -I../src -I../src-cpp
LDFLAGS += -rdynamic -L../src -L../src-cpp

# Latest Kafka version
KAFKA_VERSION?=2.3.0
# Kafka versions for compatibility tests
COMPAT_KAFKA_VERSIONS?=0.8.2.2 0.9.0.1 0.11.0.3 1.0.2 $(KAFKA_VERSION)
-include ../Makefile.config

all: $(BIN) run_par

#
# These targets spin up a cluster and runs the test suite
# with different parameters.
#

broker: $(BIN)
	./broker_version_tests.py --conf '{"parallel":1, "args":"-Q"}' $(KAFKA_VERSION)

broker_idempotent: $(BIN)
	./broker_version_tests.py --conf '{"parallel":1, "args":"-P -L -Q"}' $(KAFKA_VERSION)

non_sparse_connections: $(BIN)
	./broker_version_tests.py --brokers 5 \
		--conf '{"parallel":1, "args": "-L -Q", "sparse_connections": "false"}' $(KAFKA_VERSION)

sasl: $(BIN)
	./sasl_test.py --conf '{"parallel":1, "args":"-L -Q"}' --debug $(KAFKA_VERSION)

# Run the full test suite(s)
full: broker broker_idempotent sasl


#
# The following targets require an existing cluster running (test.conf)
#
quick:
	@echo "Running quick(er) test suite (without sockem)"
	./run-test.sh -p5 -Q -E

run_par: $(BIN)
	@echo "Running tests in parallel"
	./run-test.sh -p5

run_seq: $(BIN)
	@echo "Running tests sequentially"
	./run-test.sh -p1

run_local: $(BIN)
	@echo "Running local broker-less tests with idempotent producer"
	./run-test.sh -p5 -l -P

idempotent_par: $(BIN)
	./run-test.sh -p5 -P

idempotent_seq: $(BIN)
	./run-test.sh -p1 -P

idempotent: idempotent_par


# Delete all test topics (based on prefix)
delete_topics:
	TESTS=none ./run-test.sh -D bare

.PHONY:

build: $(BIN) interceptor_test

test.o: ../src/librdkafka.a ../src-cpp/librdkafka++.a interceptor_test



include ../mklove/Makefile.base

ifeq ($(_UNAME_S),Darwin)
interceptor_test: .PHONY
else
interceptor_test: .PHONY
	$(MAKE) -C $@
endif


tinycthread.o: ../src/tinycthread.c
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $<

tinycthread_extra.o: ../src/tinycthread_extra.c
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $<

rdlist.o: ../src/rdlist.c
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $<


clean:
	rm -f *.test $(OBJS) $(BIN)
	$(MAKE) -C interceptor_test clean

clean-output:
	# Clean test output files
	rm -f stats_*.json *.offset

realclean: clean clean-output
	rm -f test_report_*.json

0%:
	TESTS=$@ ./run-test.sh

java: .PHONY
	make -C java

# Run test-suite with ASAN
asan:
	@(echo "### Running tests with AddressSanitizer")
	(cd .. ; ./dev-conf.sh asan)
	./broker_version_tests.py $(KAFKA_VERSION)

# Run test-suite with TSAN
tsan:
	@(echo "### Running tests with ThreadSanitizer")
	(cd .. ; ./dev-conf.sh tsan)
	./broker_version_tests.py $(KAFKA_VERSION)

# Run full test-suite with a clean release build
pristine-full:
	@(echo "### Running full test-suite with clean build")
	(cd .. ; ./dev-conf.sh clean)
	make full

# Run backward compatibility tests
compat:
	@(echo "### Running compatibility tests with Apache Kafka versions $(COMPAT_KAFKA_VERSIONS)")
	./broker_version_tests.py --rdkconf '{"args": "-Q"}' \
		$(COMPAT_KAFKA_VERSIONS)

# Run a full release / PR test.
# (| is for not running suites in parallel)
release-test: | asan tsan pristine-full compat
