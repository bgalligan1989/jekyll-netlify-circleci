require 'pp'
require 'rake'


namespace :html do

	require 'shellwords'
	require 'find'
	require 'jekyll'


	desc 'Build jekyll website. Use this to have automagically tidied and compressed css and js'
	task :build_website do

		Jekyll::Commands::Build.process(profile: true)
		#we have to do the css compression after the site is built.
		#the scss files are rendered, i couldn't find a way to
		#render them manually without going through a bunch of
		#hurdles.
		if is_production?
			print "Building for Production \n"

			compress_assets type: 'js'
			compress_assets type: 'css'

			#cleanup
			FileUtils.rm_rf('_site/vendor')
			#rm 'js/main.js'
			configuration = Jekyll.configuration

			['stylesheets', 'javascripts'].each do |human_name|
				if configuration['lib'] && configuration['lib'][human_name]
					configuration['lib'][human_name].each do |asset_src|
						if File.exists? './_site/' + asset_src
							FileUtils.rm('./_site/' + asset_src)
						end
					end
				end
			end
		end
	end

	desc 'Serve jekyll website.'
	task :serve_website do

		options = {
			:background => false
		}
		option_parser = OptionParser.new
		option_parser.on('--background') do
			options[:background] = true
		end

		#jekyll serve allows you to provide an ssl cert / key pair
		#it's used by the voyager tests that are kicked off in circle.

		option_parser.on('--ssl-key=KEY') do |key|
			options[:ssl_key] = key
		end

		option_parser.on('--ssl-cert=CERT') do |cert|
			options[:ssl_cert] = cert
 		end

 		option_parser.on('--nohup') do
 			options[:nohup] = true
 		end

		#this was a headache to get working. if you look at examples on the interwebs, it's claimed that
		#rake will give you the args passed after the double dash, i.e., rake task -- --arg1 --arg2
		#should be available. this behavior has changed, so instead we have to fetch the arguments from the
		#global ARGV constant that's declared at runtime.
    	args = option_parser.order!(ARGV) {}
    	option_parser.parse!(args)


		if is_production?
			Jekyll::Commands::Build.process(profile: true)
			compress_assets type: 'js'
			compress_assets type: 'css'
		end
		jekyll_cmd = ''

		if is_production?
			jekyll_cmd = 'JEKYLL_ENV=production '
		end

		#nohup is needed so that when the jekyll serve --detach command is run
		#when circle completes the command the server process isn't sent SIGHUP.
		if options[:nohup]
			jekyll_cmd += ' nohup '
		end

		jekyll_cmd += "jekyll serve --watch "
		if (options[:background])
			jekyll_cmd += ' --detach'
		end

		if options[:ssl_key]
			jekyll_cmd += ' --ssl-key ' + options[:ssl_key]
		end

		if options[:ssl_cert]
			jekyll_cmd += ' --ssl-cert ' + options[:ssl_cert]
		end


		system jekyll_cmd
	end

	desc 'Compress javascript'
	task :compress_javascript do
		compress_assets type: 'js'
	end

	desc 'Compress CSS'
	task :compress_css do
		compress_assets type: 'css'
	end

	def is_production?
		ENV['JEKYLL_ENV'] && ENV['JEKYLL_ENV'] == 'production'
	end

	def compress_assets options = {}

		#we c
		unless Dir.exists? './_site'
			Dir.mkdir('./_site', 0655)
		end

		unless ['css', 'js'].include? options[:type]
			raise 'Unknown asset type provided: ' + options[:type]
		end

		puts "Compressing #{options[:type]}..."

		# linux
		#yui_compressor_cmd = `which yui-compressor`.strip!
		uglifyjs_cmd = `which uglifyjs`.strip!
		uglifycss_cmd = `which uglifycss`.strip!

		# mac os
		#if !yui_compressor_cmd
		#	yui_compressor_cmd = `which yuicompressor`.strip!
		#end

		#unless yui_compressor_cmd && yui_compressor_cmd.length > 0
		#	raise "Could not find the yui compressor. Either install it or ensure that it is located in one of the directories specified in the PATH environment variable."
		#end

		unless uglifyjs_cmd && uglifyjs_cmd.length > 0
			raise "Could not find UglifyJS. Either install it or ensure that it is located in one of the directories specified in the PATH environment variable."
		end

		unless uglifycss_cmd && uglifycss_cmd.length > 0
			raise "Could not find UglifyCSS. Either install it or ensure that it is located in one of the directories specified in the PATH environment variable."
		end

		#we load the jekyll config to figure out what we need to compress.
		configuration = Jekyll.configuration

		human_name = options[:type] == 'css' ? 'stylesheets' : 'javascripts'

		assets_to_compress = []
		if configuration['lib'] && configuration['lib'][human_name]
			configuration['lib'][human_name].each do |asset_to_include|
				assets_to_compress << (options[:type] == 'css' ? '_site/' : '') + Shellwords.escape(asset_to_include)
			end
		end


		if configuration['vendor'] && configuration['vendor'][human_name]
			configuration['vendor'][human_name].each do |js_script|
				assets_to_compress << (options[:type] == 'css' ? '_site/' : '') + Shellwords.escape("vendor/#{js_script}")
			end
		end

		assets_to_compress.each do |asset_file|

			if !File.exists? asset_file
				raise 'Could not find the file ' + asset_file + '.'
			end
		end

		#compression_command = sprintf("cat %s | %s --type %s", assets_to_compress.join(' '), uglify_cmd, options[:type])
		if options[:type] == 'js'
			compression_command = sprintf("uglifyjs %s --compress --mangle", assets_to_compress.join(' '))
		else
			compression_command = sprintf("uglifycss %s", assets_to_compress.join(' '))
		end

		compression_result = `#{compression_command}`

		if compression_result.length == 0
			raise 'It looks like there was a problem with compressing the ' + options[:type] + ' files.'
		end

		File.open sprintf('./_site/%s/www-min.%s', options[:type], options[:type]), 'w' do |file|
			file.write(compression_result)
		end
	end
end

namespace :test do

	require 'html-proofer'
	require 'optparse'
	require 'fileutils'

	desc 'Run html proofer against the jekyll website.'
	task :proof_website do |task, args|

		ALLOWED_OUTPUT_FORMATS = %i|junit|
		options = {}
		option_parser = OptionParser.new
		option_parser.on('--format=FORMAT', ALLOWED_OUTPUT_FORMATS) do |value|
			options[:output_format] = value
		end

		option_parser.on('--output-file=OUTPUT') do |value|
			options[:output_file] = value
		end

		args = option_parser.order!(ARGV) {}
		option_parser.parse!(args)

		html_proofer_opts = {
			:assume_extension => true,
			# :check_sri => true,
			:check_external_hash => true,
			:check_html => true,
			:check_img_http => true,
			:check_opengraph => true,
			# :enforce_https => true,
			:cache => {
			  :timeframe => '6w'
			},
			:http_status_ignore => [
				999
			],
			:url_ignore => [
				#"/users/login",
				"/",
				"#"
			]
		}

		if options[:output_format] && ! options[:output_file]
			raise "You must specify both the output file and output format."
		end

		begin
			# @fuery says:
			#   the folder shuffling and sed-replace operation below is to
			#   circumvent data-delayed-* errors from html_proofer.
			#
			#   this is b/c of some used-elswhere JS magic (not in this boilerplate repo) that translates
			#   <img src="cached-placeholder.jpg" data-delayed-src="real-image-url.png"> into the natural
			#   <img src="real-image-url.png"> tags. If you use something like this on your own site,
			#   i.e., some scheme that allows intelligent progressive loading of images,
			#   you can substiture the data-delayed-* regex as you see fit.
 			#
			#   If all of this is over your head or otherwise confusing, it is
			#   probable that this all Just Works™ anyway.
			#
			system('mkdir -p $(pwd)/_html_proofer; cp -r $(pwd)/_site/* $(pwd)/_html_proofer')
			if (/darwin/ =~ RUBY_PLATFORM) != nil
				# Mac OS
				system('find $(pwd)/_html_proofer/ -name *.html -exec sed -i "" \'s/data-delayed-//g\' {} \;')
			#elsif (/cygwin|mswin|mingw|bccwin|wince|emx/ =~ RUBY_PLATFORM) != nil
			else
				# Linux (or at least not windows and not mac :-)
				system('find $(pwd)/_html_proofer/ -name *.html -exec sed -i \'s/data-delayed-//g\' {} \;')
			end

			runner = HTMLProofer.check_directory("_html_proofer/", html_proofer_opts)
			runner.run
			FileUtils.rm_rf('_html_proofer')
		rescue => msg

			if options[:output_format] && Object.respond_to?(:"build_#{options[:output_format]}_output", true)
				self.send :"build_#{options[:output_format]}_output", runner.instance_variable_get(:@failures), output_file: options[:output_file]
			end
			FileUtils.rm_rf('_html_proofer')
			#the exit code tells circleci that the operation failed.
			exit 1

		end
	end

	#this is used to generate results that play well with circleci.
	#there are some small improvements that could be made to make it
	#more according to the junit specification, but it's really nbd.
	def build_junit_output failures, options = {}
		builder = Nokogiri::XML::Builder.new do |xml|

			xml.root do
				xml.testsuite(name: 'html-proofer', failures: failures.count) do

					failures.each do |failure|
						xml.testcase(
							classname: 'htmlproofer.website.validation',
							name:"HTML Proofing error in #{failure.path} line #{failure.line}: failure.desc",
							file:"Rakefile",
							time:0
						) do
							xml.failure("HTML Proofing error in #{failure.path} line #{failure.line}: #{failure.desc}\n#{failure.content}",
								message:"HTML Proofing error in #{failure.path} line #{failure.line}: #{failure.desc}", type:"HTMLProoferError")
						end
					end
				end
			end
		end

		if options[:output_file]
			File.open(options[:output_file], 'w') do |file|
				file.write(builder.to_xml)
			end

		end
	end
end
