TESTS := $(wildcard *.hex)

all:

run:
	$(foreach test, $(TESTS), timeout 2s ../tinyrv -s $(test) || exit;)

clean: