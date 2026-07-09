---
layout: default
title: Contact
permalink: /contact/
---

## Would you like to get in contact?

Feel free to reach out to me on my socials:

{% if site.social_links %}

<ul aria-label="Social links">
    {% for social in site.social_links %}
    <li>
    <a href="{{ social.url }}" class="social-link" target="_blank" rel="noopener noreferrer">
    {% include social-icon.html icon=social.icon %}
    <span>{{ social.name }}</span>
    </a>
    </li>
    {% endfor %}
</ul>
{% endif %}

If email is your preferred method of communication, you can reach me at:

{% if site.emails %}

<ul class="email-list" aria-label="Emails">
    {% for email in site.emails %}
    {% assign encoded_email = email.address | split: '' | reverse | join: '' %}
    <li>
    <span class="email-list__label">{{ email.name }}:</span>
    <a class="social-link email-list__address" href="#" data-email-reversed="{{ encoded_email }}">
        <span class="email-list__text" aria-hidden="true"></span>
    </a>
    <button type="button" class="email-list__copy" hidden>Copy</button>
    <span class="email-list__status" aria-live="polite"></span>
    </li>
    {% endfor %}
</ul>

<script>
document.addEventListener('DOMContentLoaded', function () {
    function copyText(text, statusNode) {
        function markCopied() {
            statusNode.textContent = 'Copied';
            window.setTimeout(function () {
                statusNode.textContent = '';
            }, 1500);
        }

        if (navigator.clipboard && navigator.clipboard.writeText) {
            navigator.clipboard.writeText(text).then(markCopied).catch(function () {
                fallbackCopy(text, statusNode, markCopied);
            });
            return;
        }

        fallbackCopy(text, statusNode, markCopied);
    }

    function fallbackCopy(text, statusNode, done) {
        var input = document.createElement('input');
        input.value = text;
        input.setAttribute('readonly', 'readonly');
        input.style.position = 'fixed';
        input.style.left = '-9999px';
        document.body.appendChild(input);
        input.select();

        try {
            document.execCommand('copy');
            done();
        } catch (error) {
            statusNode.textContent = 'Copy failed';
        }

        document.body.removeChild(input);
    }

    document.querySelectorAll('[data-email-reversed]').forEach(function (link) {
        var reversed = link.getAttribute('data-email-reversed');
        var email = reversed.split('').reverse().join('');
        var listItem = link.closest('.email-list li');
        var textNode = link.querySelector('.email-list__text');
        var copyButton = listItem.querySelector('.email-list__copy');
        var statusNode = listItem.querySelector('.email-list__status');

        link.href = 'mailto:' + email;
        textNode.textContent = email;
        copyButton.hidden = false;

        copyButton.addEventListener('click', function () {
            copyText(email, statusNode);
        });
    });
});
</script>

{% endif %}
