require 'pp'
require_relative 'lib/colors'
include Colors

task :default => :check

REPO_DIR = File.dirname(__FILE__)
SAS_DIR = "#{REPO_DIR}/vendor/sas"

OUT = ENV['OUT'] || "#{REPO_DIR}/out"

#######
# Other environment variables that may be set by the user for specific tasks (see README.md)
#######
STRAIN_NAME = ENV['STRAIN_NAME']
SPECIES = ENV['SPECIES']

#############################################################
#  IMPORTANT!
#  This Rakefile runs with the working directory set to OUT
#  All filenames from hereon are relative to that directory
#############################################################
Dir.chdir(OUT)

task :env do
  puts "Output directory: #{OUT}"
  mkdir_p OUT
  mkdir_p File.join(REPO_DIR, "vendor")
  
  sc_orga_scratch = "/sc/orga/scratch/#{ENV['USER']}"
  ENV['TMP'] ||= Dir.exists?(sc_orga_scratch) ? sc_orga_scratch : "/tmp"
  ENV['PERL5LIB'] ||= "/usr/bin/perl5.10.1"
end

file "#{REPO_DIR}/scripts/env.sh" => "#{REPO_DIR}/scripts/env.example.sh" do
  cp "#{REPO_DIR}/scripts/env.example.sh", "#{REPO_DIR}/scripts/env.sh"
end

desc "Checks environment variables and requirements before running tasks"
task :check => ["#{REPO_DIR}/scripts/env.sh", :env] do
  env_error = "Configure this in scripts/env.sh and run `source scripts/env.sh` before running rake."
  unless `module avail 2>&1 | grep smrtpipe/2.2.0` != ''
    abort "You must have the smrtpipe/2.2.0 module in your MODULEPATH."
  end
  unless ENV['SMRTPIPE'] && File.exists?("#{ENV['SMRTPIPE']}/example_params.xml")
    abort "SMRTPIPE must be set to the directory containing example_params.xml for smrtpipe.py.\n#{env_error}"
  end
  unless ENV['SMRTANALYSIS'] && File.exists?("#{ENV['SMRTANALYSIS']}/etc/setup.sh")
    abort <<-ERRMSG
      SMRTANALYSIS must be set to the ROOT directory for the SMRT Analysis package, v2.2.0.
      This software can be downloaded from http://www.pacb.com/devnet/
      #{env_error}"
    ERRMSG
  end
end

# pulls down http://blog.theseed.org/downloads/sas.tgz --> ./vendor/sas
#   then it adds SAS libs to PERL5LIB
#   then it adds SAS bins to PATH
task :sas => [:env, "#{SAS_DIR}/sas.tgz", "#{SAS_DIR}/modules/lib"] do
  ENV['PERL5LIB'] = "#{ENV['PERL5LIB']}:#{SAS_DIR}/lib:#{SAS_DIR}/modules/lib"
  ENV['PATH'] = "#{SAS_DIR}/bin:#{ENV['PATH']}"
end

directory SAS_DIR
file "#{SAS_DIR}/sas.tgz" => [SAS_DIR] do |t|
  Dir.chdir(File.dirname(t.name)) do
    system("curl -O 'http://blog.theseed.org/downloads/sas.tgz'")
    system("tar xvzf sas.tgz")
  end
end

directory "#{SAS_DIR}/modules/lib"
file "#{SAS_DIR}/modules/lib" => ["#{SAS_DIR}/sas.tgz"] do |t|
  Dir.chdir("#{SAS_DIR}/modules") do
    system("./BUILD_MODULES")
  end
end


# =======================
# = pull_down_raw_reads =
# =======================

desc "Uses scripts/ccs_get.py to save raw reads from PacBio to OUT directory"
task :pull_down_raw_reads => [:check, "bash5.fofn"]  # <-- file(s) created by this task
file "bash5.fofn" do |t, args|                       # <-- implementation for generating each of these files
  job_id = ENV['SMRT_JOB_ID'] # an example that works is 019194
  abort "FATAL: Task pull_down_raw_reads requires specifying SMRT_JOB_ID" unless job_id
  
  system <<-SH
    python #{REPO_DIR}/scripts/ccs_get.py --noprefix -e bax.h5 #{job_id} -i &&
    find #{OUT}/*bax.h5 > bash5.fofn
  SH
  # NOTE: we will change the above to not fetch the full sequence, but rather symlink to it on minerva, like so
  #       we could even skip straight to circularize_assembly if the polished_assembly.fasta is already there
  # cp /sc/orga/projects/pacbio/userdata_permanent/jobs/#{job_id[0..3]}/#{job_id}/input.fofn baxh5.fofn
  # mkdir_p "data"
  # ln -s /sc/orga/projects/pacbio/userdata_permanent/jobs/data/#{job_id[0..3]}/#{job_id}/polished_assembly.fasta <input assembly file name>

end

# ======================
# = assemble_raw_reads =
# ======================

desc "Uses smrtpipe.py to assemble raw reads from PacBio within OUT directory"
task :assemble_raw_reads => [:check, "data/polished_assembly.fasta.gz"]
file "data/polished_assembly.fasta.gz" => "bash5.fofn" do |t|
  system <<-SH
    module load smrtpipe/2.2.0
    source #{ENV['SMRTANALYSIS']}/etc/setup.sh &&
    fofnToSmrtpipeInput.py bash5.fofn > bash5.xml &&
    cp #{ENV['SMRTPIPE']}/example_params.xml \. &&
    smrtpipe.py -D TMP=#{ENV['TMP']} -D SHARED_DIR=#{ENV['SHARED_DIR']} -D NPROC=16 -D CLUSTER=LSF -D MAX_THREADS=16 --distribute --params example_params.xml xml:bash5.xml 
  SH
end

# ========================
# = circularize_assembly =
# ========================

desc "Circularizes the PacBio assembly"
task :circularize_assembly => [:check, "data/polished_assembly_circularized.fasta"]
file "data/polished_assembly_circularized.fasta" => "data/polished_assembly.fasta.gz" do |t|
  system "gunzip -c data/polished_assembly.fasta.gz >data/polished_assembly.fasta" and
  system "#{REPO_DIR}/scripts/circularizeContig.pl data/polished_assembly.fasta"
end

# =======================
# = resequence_assembly =
# =======================

desc "Resequences the circularized assembly"
task :resequence_assembly => [:check, "data/#{STRAIN_NAME}_consensus.fasta"]
file "data/#{STRAIN_NAME}_consensus.fasta" => "data/polished_assembly_circularized.fasta" do |t|
  abort "FATAL: Task resequence_assembly requires specifying STRAIN_NAME" unless STRAIN_NAME 
  
  mkdir_p "circularized_sequence"
  system <<-SH or abort
    module load smrtpipe/2.2.0
    source #{ENV['SMRTANALYSIS']}/etc/setup.sh &&
    referenceUploader -c -p circularized_sequence -n #{STRAIN_NAME} -f data/polished_assembly_circularized.fasta
  SH
  cp "#{ENV['SMRTPIPE']}/resequence_example_params.xml", OUT
  system "perl #{REPO_DIR}/scripts/changeResequencingDirectory.pl resequence_example_params.xml " +
      "#{OUT} circularized_sequence/#{STRAIN_NAME} > resequence_params.xml" and
  system <<-SH or abort
    module load smrtpipe/2.2.0
    source #{ENV['SMRTANALYSIS']}/etc/setup.sh &&
    samtools faidx circularized_sequence/#{STRAIN_NAME}/sequence/#{STRAIN_NAME}.fasta &&
    smrtpipe.py -D TMP=#{ENV['TMP']} -D SHARED_DIR=#{ENV['SHARED_DIR']} -D NPROC=16 -D CLUSTER=LSF -D MAX_THREADS=16 --distribute --params resequence_params.xml xml:bash5.xml &&
    gunzip data/consensus.fasta.gz
  SH
  cp "data/consensus.fasta", "data/#{STRAIN_NAME}_consensus.fasta"
end


# =================
# = rast_annotate =
# =================

desc "Submits the circularized assembly to RAST for annotations"
task :rast_annotate => [:check, "data/#{STRAIN_NAME}_consensus_rast.fna", 
    "data/#{STRAIN_NAME}_consensus_rast.gbk", "data/#{STRAIN_NAME}_consensus_rast_aa.fa"]

file "data/#{STRAIN_NAME}_consensus_rast.gbk" => [:sas, "data/#{STRAIN_NAME}_consensus.fasta"] do |t|
  abort "FATAL: Task rast_annotate requires specifying STRAIN_NAME" unless STRAIN_NAME 
  abort "FATAL: Task rast_annotate requires specifying SPECIES" unless SPECIES 
  
  rast_job = %x[
    perl #{REPO_DIR}/scripts/svr_submit_status_retrieve.pl --user oattie --passwd sessiz_ev \
        --fasta data/#{STRAIN_NAME}_consensus.fasta --domain Bacteria --bioname "#{SPECIES} #{STRAIN_NAME}" \
        --genetic_code 11 --gene_caller rast
  ]
  system "perl #{REPO_DIR}/scripts/test_server.pl oattie sessiz_ev genbank #{rast_job}"
  sleep 120
  system "svr_retrieve_RAST_job oattie sessiz_ev #{rast_job} genbank > data/#{STRAIN_NAME}_consensus_rast.gbk"
end

file "data/#{STRAIN_NAME}_consensus_rast_aa.fa" => "data/#{STRAIN_NAME}_consensus_rast.gbk" do |t|
  abort "FATAL: Task rast_annotate requires specifying STRAIN_NAME" unless STRAIN_NAME 
  system <<-SH
    python #{REPO_DIR}/scripts/gb_to_fasta.py -i data/#{STRAIN_NAME}_consensus_rast.gbk -s aa \
        -o data/#{STRAIN_NAME}_consensus_rast_aa.fa
  SH
end

file "data/#{STRAIN_NAME}_consensus_rast.fna" => "data/#{STRAIN_NAME}_consensus_rast.gbk" do |t|
  abort "FATAL: Task rast_annotate requires specifying STRAIN_NAME" unless STRAIN_NAME 
  system <<-SH
    python #{REPO_DIR}/scripts/gb_to_fasta.py -i data/#{STRAIN_NAME}_consensus_rast.gbk -s nt \
        -o data/#{STRAIN_NAME}_consensus_rast.fna
  SH
end
