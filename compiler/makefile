OUTFILE = stabc.jar
CLASSPATH = "../bin/asm-8.0.1.jar;../bin/stabrt.jar;../bin/stabal.jar"
RESOURCES = resources
SFLAGS = -cp:$(CLASSPATH) -res:$(RESOURCES) -out:$(OUTFILE) -mf:MANIFEST.MF
SC = java -jar ../bin/stabc.jar

CLASSES = $(wildcard src/*/*.stab)

default: build

build:
	$(SC) $(SFLAGS) $(CLASSES)

clean:
	$(RM) $(OUTFILE)
