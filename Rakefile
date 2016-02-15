# encoding: UTF-8
require "rubygems"
require "tmpdir"
require "bundler/setup"
require "jekyll"

# Change your GitHub reponame
GITHUB_REPONAME = "github_username/repo_name"
GITHUB_REPO_BRANCH = "master"

task default: %w[publish]

desc "Generate blog files"
task :generate do
  Jekyll::Site.new(Jekyll.configuration({
    "source"      => "source/",
    "destination" => "_site",
    "config"      => "_config.yml"
  })).process
end

desc "Generate and publish blog to gh-pages"
task :publish => [:generate] do
  Dir.mktmpdir do |tmp|
    cp_r "_site/.", tmp

    pwd = Dir.pwd
    Dir.chdir tmp

    system "git init"
    system "git checkout --orphan #{GITHUB_REPO_BRANCH}"
    system "git add ."
    message = "Site updated at #{Time.now.utc}"
    system "git commit -am #{message.inspect}"
    system "git remote add origin git@github.com:#{GITHUB_REPONAME}.git"
    system "git push origin #{GITHUB_REPO_BRANCH} --force"

    Dir.chdir pwd
  end
end
