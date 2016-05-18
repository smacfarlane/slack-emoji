require 'fileutils'
require 'dotenv'
require 'yaml'
require 'open-uri'
require 'rest-client'

Dotenv.load
URL = "https://#{ENV['SLACK_TEAM']}.slack.com".freeze

task default: [:cache, :upload]

desc 'Upload emojis described in yaml'
task :upload do
  Rake::Task['cache'].invoke
  fail "Missing SLACK_TEAM" unless ENV['SLACK_TEAM']

  cookies = if ENV['SLACK_COOKIES']
    env_cookies
  else
    slack_login.cookies
  end
  crumb = fetch_crumb("#{URL}/customize/emoji", cookies: cookies)

  emoji_list = fetch_emojis("#{URL}/customize/emoji", cookies: cookies)

  emoji_packs.each do |pack|
    pack['emojis'].each do |emoji|
      if has_emoji?(emoji_list, emoji['name'])
        puts "Already uploaded: #{pack['title']} - #{emoji['name']}"
        next
      end
      if !File.exist?(file_cache_path(pack['title'], emoji['name']))
        puts "Source file #{file_cache_path(pack['title'], emoji['name'])} not found. Ensure it cached correctly"
        next
      end

      puts "Uploading #{pack['title']} - #{emoji['name']}"
      params = {
        add: 1,
        crumb: crumb,
        name: emoji['name'],
        mode: 'data',
        img: File.new(file_cache_path(pack['title'], emoji['name']), 'rb')
      }
      response = slack_post("#{URL}/customize/emoji", params, cookies: cookies)
      sleep 1
    end
  end
end

task :cache do
  require 'net/http'
  # Fetch and cache emoji
  emoji_packs.each do |pack|
    pack_cache = cache_path(pack['title'])
    FileUtils.mkdir_p(pack_cache) unless File.exist?(pack_cache)
    puts "Caching #{pack['title']}... "
    pack['emojis'].each do |emoji|
      cached_file = file_cache_path(pack['title'], emoji['name'])
      unless File.exist?(cached_file)
        uri = URI.parse(emoji['src'])
        res = Net::HTTP.get_response(uri)
        if res.is_a?(Net::HTTPSuccess)
          File.open(cached_file, 'wb') { |f| f << res.body }
        else
          puts "Failed to fetch #{emoji['src']}"
        end
      end
    end
  end
end

def emoji_packs
  Dir.glob("packs/*.yaml").map { |f| YAML.load_file(f) }
end

def fetch_crumb(url, headers={})
  require 'nokogiri'
  html = RestClient.get(url, headers)
  emoji = Nokogiri::HTML(html.body)

  emoji.css('input[name="crumb"]').first["value"]
end

def cache_path(name)
  File.join("cache", name)
end

def file_cache_path(pack, name)
  File.join(cache_path(pack), name)
end

def slack_post(url, params, headers={})
  RestClient.post(url, params, headers) do |response, request, result, &block|
    case response.code
    when 302
      response
    else
      response.return!(request, result, &block)
    end
  end
end

def slack_login
  require 'highline'
  email = HighLine.new.ask("Email: ")
  password = HighLine.new.ask("Password: ") { |q| q.echo = "x" }
  params = {
    email: email,
    password: password,
    signin: 1,
    crumb: fetch_crumb(URL)
  }

  response = slack_post(URL, params)
  puts "export SLACK_COOKIES=\"#{response.cookies.map{|k,v| "#{k}=#{v}"}.join("; ")}\""
  response
end

def env_cookies
  ENV['SLACK_COOKIES'].split(";").each_with_object({}) do |cookie, obj|
    k, v = cookie.split("=")
    obj[k.strip.to_sym] = v.strip
  end
end

def fetch_emojis(url, headers={})
  html = RestClient.get(url, headers)
  Nokogiri::HTML(html.body).css('tr.emoji_row').map do |row|
    row.xpath('./td')[1].text.strip
  end
end

def has_emoji?(list, emoji)
  list.include?(":#{emoji}:")
end