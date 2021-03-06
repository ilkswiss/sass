require 'rubygems/package'

# ----- Utility Functions -----

def scope(path)
  File.join(File.dirname(__FILE__), path)
end

# ----- Default: Testing ------

task :default => :test

require 'rake/testtask'

LINE_SIZE = 80
DECORATION_CHAR = '#'

def print_header(string)
  length = string.length
  puts DECORATION_CHAR * LINE_SIZE
  puts string.center(length + 2, ' ').center(LINE_SIZE, DECORATION_CHAR)
  puts DECORATION_CHAR * LINE_SIZE
end

desc "Run all tests"
task :test do
  test_cases = [
    {
      'env'   => {'MATHN' => 'true'},
      'tasks' => ['test:ruby', 'test:spec', :rubocop]
    },
    {
      'env'   => {'MATHN' => 'false'},
      'tasks' => ['test:ruby']
    }
  ]

  test_cases.each do |test_case|
    env = test_case['env']
    tasks = test_case['tasks']

    env.each do |key, value|
      ENV[key] = value
    end
    tasks.each do |task|
      print_header("Running task: #{task}, env: #{env}")
      Rake::Task[task].execute
    end
  end
end

namespace :test do
  desc "Run the ruby tests (without sass-spec)"
  Rake::TestTask.new("ruby") do |t|
    t.libs << 'test'
    test_files = FileList[scope('test/**/*_test.rb')]
    test_files.exclude(scope('test/rails/*'))
    test_files.exclude(scope('test/plugins/*'))
    t.test_files = test_files
    t.warning = true
    t.verbose = true
  end

  desc "Run sass-spec tests against the local code."
  task :spec do
    require "yaml"
    sass_spec_options = YAML.load_file(scope("test/sass-spec.yml"))
    enabled = sass_spec_options.delete(:enabled)
    unless enabled
      puts "SassSpec tests are disabled."
      next
    end
    if ruby_version_at_least?("1.9.2")
      old_load_path = $:.dup
      begin
        $:.unshift(File.join(File.dirname(__FILE__), "lib"))
        begin
          require 'sass_spec'
        rescue LoadError
          puts "You probably forgot to run: bundle exec rake"
          raise
        end
        default_options = {
          :spec_directory => SassSpec::SPEC_DIR,
          :engine_adapter => SassEngineAdapter.new,
          :generate => false,
          :tap => false,
          :skip => false,
          :verbose => false,
          :filter => "",
          :limit => -1,
          :unexpected_pass => false,
          :nuke => false,
        }
        SassSpec::Runner.new(default_options.merge(sass_spec_options)).run || exit(1)
      ensure
        $:.replace(old_load_path)
      end
    else
      "Skipping sass-spec on ruby versions less than 1.9.2"
    end
  end
end

# ----- Code Style Enforcement -----

def ruby_version_at_least?(version_string)
  ruby_version = Gem::Version.new(RUBY_VERSION.dup)
  version = Gem::Version.new(version_string)
  ruby_version >= version
end

begin
  require 'rubocop/rake_task'
  RuboCop = Rubocop unless defined?(RuboCop)
  RuboCop::RakeTask.new do |t|
    t.patterns = FileList["lib/**/*"]
  end
rescue LoadError
  task :rubocop do
    puts "Rubocop is disabled."
    puts "Passing this check is required in order for your patch to be accepted."
    puts "Install Rubocop and then run the style check with: rake rubocop."
  end
end

# ----- Packaging -----

# Don't use Rake::GemPackageTast because we want prerequisites to run
# before we load the gemspec.
desc "Build all the packages."
task :package => [:revision_file, :date_file, :permissions] do
  version = get_version
  File.open(scope('VERSION'), 'w') {|f| f.puts(version)}
  load scope('sass.gemspec')
  Gem::Package.build(SASS_GEMSPEC)
  sh %{git checkout VERSION}

  pkg = "#{SASS_GEMSPEC.name}-#{SASS_GEMSPEC.version}"
  mkdir_p "pkg"
  verbose(true) {mv "#{pkg}.gem", "pkg/#{pkg}.gem"}

  sh %{rm -f pkg/#{pkg}.tar.gz}
  verbose(false) {SASS_GEMSPEC.files.each {|f| sh %{tar rf pkg/#{pkg}.tar #{f}}}}
  sh %{gzip pkg/#{pkg}.tar}
end

task :permissions do
  sh %{chmod -R a+rx bin}
  sh %{chmod -R a+r .}
  require 'shellwords'
  Dir.glob('test/**/*_test.rb') do |file|
    next if file =~ %r{^test/haml/spec/}
    sh %{chmod a+rx #{file}}
  end
end

task :revision_file do
  require scope('lib/sass')

  release = Rake.application.top_level_tasks.include?('release') || File.exist?(scope('EDGE_GEM_VERSION'))
  if Sass.version[:rev] && !release
    File.open(scope('REVISION'), 'w') { |f| f.puts Sass.version[:rev] }
  elsif release
    File.open(scope('REVISION'), 'w') { |f| f.puts "(release)" }
  else
    File.open(scope('REVISION'), 'w') { |f| f.puts "(unknown)" }
  end
end

task :date_file do
  File.open(scope('VERSION_DATE'), 'w') do |f|
    f.puts Time.now.utc.strftime('%d %B %Y %T %Z')
  end
end

# We also need to get rid of this file after packaging.
at_exit do
  File.delete(scope('REVISION')) rescue nil
  File.delete(scope('VERSION_DATE')) rescue nil
end

desc "Install Sass as a gem. Use SUDO=1 to install with sudo."
task :install => [:package] do
  gem  = RUBY_PLATFORM =~ /java/  ? 'jgem' : 'gem'
  sh %{#{'sudo ' if ENV["SUDO"]}#{gem} install --no-ri pkg/sass-#{get_version}}
end

desc "Release a new Sass package to RubyGems.org."
task :release => [:check_release, :package] do
  version = File.read(scope("VERSION")).strip
  sh %{gem push pkg/sass-#{version}.gem}
end

# Ensures that the VERSION file has been updated for a new release.
task :check_release do
  version = File.read(scope("VERSION")).strip
  raise "There have been changes since current version (#{version})" if changed_since?(version)
  raise "VERSION_NAME must not be 'Bleeding Edge'" if File.read(scope("VERSION_NAME")) == "Bleeding Edge"
end

# Reads a password from the command line.
#
# @param name [String] The prompt to use to read the password
def read_password(prompt)
  require 'readline'
  system "stty -echo"
  Readline.readline("#{prompt}: ").strip
ensure
  system "stty echo"
  puts
end

# Returns whether or not the repository, or specific files,
# has/have changed since a given revision.
#
# @param rev [String] The revision to check against
# @param files [Array<String>] The files to check.
#   If this is empty, checks the entire repository
def changed_since?(rev, *files)
  IO.popen("git diff --exit-code #{rev} #{files.join(' ')}") {}
  return !$?.success?
end

# Get the version string. If this is being installed from Git,
# this includes the proper prerelease version.
def get_version
  File.read(scope('VERSION').strip)
end

task :watch_for_update do
  sh %{ruby extra/update_watch.rb}
end

# ----- Documentation -----

task :rdoc do
  puts '=' * 100, <<END, '=' * 100
Sass uses the YARD documentation system (http://github.com/lsegal/yard).
Install the yard gem and then run "rake doc".
END
end

begin
  require 'yard'

  namespace :doc do
    task :sass do
      require scope('lib/sass')
      Dir[scope("yard/default/**/*.sass")].each do |sass|
        File.open(sass.gsub(/sass$/, 'css'), 'w') do |f|
          f.write(Sass::Engine.new(File.read(sass)).render)
        end
      end
    end

    desc "List all undocumented methods and classes."
    task :undocumented do
      opts = ENV["YARD_OPTS"] || ""
      ENV["YARD_OPTS"] = opts.dup + <<OPTS
 --list --tag comment --hide-tag comment --query "
  object.docstring.blank? &&
  !(object.type == :method && object.is_alias?)"
OPTS
      Rake::Task['yard'].execute
    end

    # Add IDs to the reference to match the old IDs generated by Maruku. This
    # ensures that old anchored links continue to work.
    task :add_ids do
      require 'nokogiri'

      doc = Nokogiri::HTML(File.read('doc/file.SASS_REFERENCE.html'))

      doc.css("#filecontents").css("h1, h2, h3, h4, h5, h6").each do |h|
        next if h.inner_text.empty?
        h['id'] =
          case h.inner_text
          when "Referencing Parent Selectors: &"; "parent-selector"
          when /^Comments:/; "comments"
          when "Strings"; "sass-script-strings"
          when "Division and /"; "division-and-slash"
          when /^Subtraction,/; "subtraction"
          when "& in SassScript"; "parent-script"
          when "@-Rules and Directives"; "directives"
          when "@extend-Only Selectors"; "placeholders"
          when "@extend-Only Selectors"; "placeholders"
          when "@each"; "each-directive"
          when "Multiple Assignment"; "each-multi-assign"
          when "Mixin Directives"; "mixins"
          when /^Defining a Mixin:/; "defining_a_mixin"
          when /^Including a Mixin:/; "including_a_mixin"
          when "Arguments"; "mixin-arguments"
          when "Passing Content Blocks to a Mixin"; "mixin-content"
          else
            h.inner_text.downcase.gsub(/[^a-z _-]/, '').gsub(' ', '_')
          end
      end

      # Give each option an anchor.
      doc.css("#filecontents li p strong code").each do |c|
        c['id'] = c.inner_text.gsub(/:/, '') + '-option'
      end

      File.write('doc/file.SASS_REFERENCE.html', doc.to_html)
    end
  end

  YARD::Rake::YardocTask.new do |t|
    t.files = FileList.new(scope('lib/**/*.rb')) do |list|
      list.exclude('lib/sass/plugin/merb.rb')
      list.exclude('lib/sass/plugin/rails.rb')
    end.to_a
    t.options << '--incremental' if Rake.application.top_level_tasks.include?('redoc')
    t.options += FileList.new(scope('yard/*.rb')).to_a.map {|f| ['-e', f]}.flatten
    files = FileList.new(scope('doc-src/*')).to_a.sort_by {|s| s.size} + %w[MIT-LICENSE VERSION]
    t.options << '--files' << files.join(',')
    t.options << '--template-path' << scope('yard')
    t.options << '--title' << ENV["YARD_TITLE"] if ENV["YARD_TITLE"]

    t.before = lambda do
      if ENV["YARD_OPTS"]
        require 'shellwords'
        t.options.concat(Shellwords.shellwords(ENV["YARD_OPTS"]))
      end
    end
  end
  Rake::Task['yard'].prerequisites.insert(0, 'doc:sass')
  Rake::Task['yard'].instance_variable_set('@comment', nil)

  desc "Generate Documentation"
  task :doc => [:yard, 'doc:add_ids']
  task :redoc => [:yard, 'doc:add_ids']
rescue LoadError
  desc "Generate Documentation"
  task :doc => :rdoc
  task :yard => :rdoc
end

# ----- Coverage -----

begin
  require 'rcov/rcovtask'

  Rcov::RcovTask.new do |t|
    t.test_files = FileList[scope('test/**/*_test.rb')]
    t.rcov_opts << '-x' << '"^\/"'
    if ENV['NON_NATIVE']
      t.rcov_opts << "--no-rcovrt"
    end
    t.verbose = true
  end
rescue LoadError; end

# ----- Profiling -----

begin
  require 'ruby-prof'

  desc <<END
Run a profile of sass.
  TIMES=n sets the number of runs. Defaults to 1000.
  FILE=str sets the file to profile. Defaults to 'complex'.
  OUTPUT=str sets the ruby-prof output format.
    Can be Flat, CallInfo, or Graph. Defaults to Flat. Defaults to Flat.
END
  task :profile do
    times  = (ENV['TIMES'] || '1000').to_i
    file   = ENV['FILE']

    require 'lib/sass'

    file = File.read(scope("test/sass/templates/#{file || 'complex'}.sass"))
    result = RubyProf.profile { times.times { Sass::Engine.new(file).render } }

    RubyProf.const_get("#{(ENV['OUTPUT'] || 'Flat').capitalize}Printer").new(result).print
  end
rescue LoadError; end
