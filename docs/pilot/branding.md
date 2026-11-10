# BoloDesk pilot branding

The India pilot uses BoloDesk as its installation and brand name, with a teal
speech-bubble icon and light/dark wordmarks in `public/brand-assets/bolodesk-*.svg`.
Both installation configuration files carry the same defaults so Enterprise's
community-plan reconciliation preserves the pilot branding.

New installations pick up these defaults during database setup. Existing
installations retain saved configuration. To apply only the pilot branding,
run the following from this branch in the target deployment:

```sh
bundle exec rails runner '
keys = %w[INSTALLATION_NAME BRAND_NAME LOGO LOGO_DARK LOGO_THUMBNAIL DISPLAY_MANIFEST]
ConfigLoader.new.general_configs.select { |config| keys.include?(config.fetch("name")) }.each do |config|
  InstallationConfig.find_by!(name: config.fetch("name")).update!(value: config.fetch("value"))
end
'
```

For local development, initialize rbenv before running Bundler. Restart the web
and worker processes after applying the configuration and reload the browser.

`DISPLAY_MANIFEST` is disabled to suppress the bundled Chatwoot manifest and
platform icons; the configured BoloDesk favicon remains available. This pilot
does not supply installable-app metadata or custom Apple touch icons.

Brand destination, terms, and privacy URLs still use their existing values.
Configure those separately once the pilot's domain and policies are ready.

To review, open the login page in light and dark modes, then sign in and check
the sidebar logo, browser title, favicon, and welcome text. Check widget and
email branding on the target deployment as well.
