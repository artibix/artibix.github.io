---
title: HexoPublisher
category:
  - Blog
date: 2024-10-31 12:54:09.159Z
updated: 2024-12-24 04:52:01.113Z
---

class HexoPublisher extends api.NoteContextAwareWidget {
    constructor() {
        super();
        this.bindMethods();
    }

    get parentWidget() {
        return 'center-pane';
    }
    doRender() {
        this.$widget = $();
        return this.$widget;
    }
    bindMethods() {
        this.previewContent = this.previewContent.bind(this);
        this.publishContent = this.publishContent.bind(this);
    }

    async refreshWithNote() {
        $(document).ready(() => {
            this.addButtonsIfNeeded();
            this.bindEventHandlers();
        });
    }

    addButtonsIfNeeded() {
        const $noteSplit = $("div.component.note-split:not(.hidden-ext)");
        if (!$noteSplit.find("div.ribbon-tab-title").hasClass('github-publisher-button')) {
            const buttonsHtml = this.getButtonsHtml();
            $noteSplit.find(".ribbon-tab-title:not(.backToHis)").last().after(buttonsHtml);
        }
    }

    getButtonsHtml() {
        return `
        	<div class="github-publisher-button ribbon-tab-spacer"></div>
            <div class="ribbon-tab-title" data-ribbon-component-name="markdownPreview">
                <span class="ribbon-tab-title-icon bx bx-file" data-title="预览文章" data-toggle-command="previewMarkdown"></span>
                <span class="ribbon-tab-title-label">预览</span>
            </div>
            <div class="github-publisher-button ribbon-tab-spacer"></div>
            <div class="ribbon-tab-title" data-ribbon-component-name="githubPublish">
                <span class="ribbon-tab-title-icon bx bx-upload" data-title="发布到GitHub" data-toggle-command="publishToGithub"></span>
                <span class="ribbon-tab-title-label">发布</span>
            </div>
        `;
    }

    bindEventHandlers() {
        const $buttons = $('div.component.note-split:not(.hidden-ext)');
        $buttons.find('[data-toggle-command="previewMarkdown"]').off("click").on("click", this.previewContent);
        $buttons.find('[data-toggle-command="publishToGithub"]').off("click").on("click", this.publishContent);
    }

    async previewContent() {
        try {
            const { content, frontMatter, note } = await this.collectNoteData();

            const processedContent = await this.processContent(content, frontMatter, note);

            const previewMessage = `预览内容（图片将会被上传到 source/images 目录）：\n\n${processedContent}`;

            api.showMessage(previewMessage);
        } catch (error) {
            api.showError('Failed to preview: ' + error.message);
        }
    }


    async publishContent() {
        try {
            const { content, frontMatter, note } = await this.collectNoteData();
            await this.validatePublishRequirements(note);

            const fileName = this.generateFileName(note);
            const processedContent = await this.processContent(content, frontMatter, note);

            const result = await this.uploadToGithub(note, fileName, processedContent, false);
            this.showPublishResult(result);
        } catch (error) {
            api.showError('Failed to publish: ' + error.message);
        }
    }

    async collectNoteData() {
        const editor = await api.getActiveContextTextEditor();
        const note = await api.getActiveContextNote();
        const content = note.getContent();
        api.showMessage(JSON.stringify(content));
        const frontMatter = await this.collectFrontMatter(note);
        return { content, frontMatter, note };
    }

    validatePublishRequirements(note) {
        const repoPathAttr = note.getAttribute('label', 'githubRepo');
        const tokenAttr = note.getAttribute('label', 'githubToken');

        if (!repoPathAttr || !tokenAttr) {
            throw new Error('Missing required attributes: githubRepo or githubToken');
        }
        return { repoPath: repoPathAttr.value, token: tokenAttr.value };
    }

    generateFileName(note) {
        return `source/_posts/${note.noteId.replace(/[^a-zA-Z0-9-]/g, '-').toLowerCase()}.md`;
    }

    async processContent(content, frontMatter, note) {
        const processedContent = await this.processImages(content, note);
        return this.generateHexoContent(processedContent, frontMatter);
    }

    async processImages(content, note) {
        const markdownRegex = /!\[([^\]]*)\]\(([^)]+)\)/g;
        const htmlImgRegex = /<img[^>]+src="([^"]+)"[^>]*>/g;
        let processedContent = content;
        // 处理 Markdown 格式的图片
        let mdMatch;
        while ((mdMatch = markdownRegex.exec(content)) !== null) {
            api.showMessage(htmlMatch);

            const [fullMatch, altText, imagePath] = mdMatch;
            processedContent = await this.handleImageUpload(
                processedContent,
                imagePath,
                fullMatch,
                note,
                (newPath) => `![${altText}](${newPath})`
            );
        }

        // 处理 HTML 格式的图片
        let htmlMatch;
        while ((htmlMatch = htmlImgRegex.exec(content)) !== null) {
            const [fullImg, imagePath] = htmlMatch;
            processedContent = await this.handleImageUpload(
                processedContent,
                imagePath,
                fullImg,
                note,
                (newPath) => {
                    // 保持原有的 HTML 标签属性，只更新 src
                    return fullImg.replace(imagePath, newPath);
                }
            );
        }

        return processedContent;
    }

    async handleImageUpload(content, imagePath, originalText, note, replacer) {
        if (imagePath.includes('api/attachments/')) {
            try {
                const matches = imagePath.match(/api\/attachments\/([^\/]+?)(\/|$)/);
                const attachmentId = matches ? matches[1] : null;
                if (!attachmentId) {
                    throw new Error('Could not extract attachment ID from path');
                }
                const attachmentUrl = `api/attachments/${attachmentId}/download`;

                const response = await fetch(attachmentUrl);
                const arrayBuffer = await response.arrayBuffer();
                const base64Content = Buffer.from(arrayBuffer).toString('base64');
                const originalFileName = imagePath.split('/').pop();

                const fileName = `source/images/${note.noteId}-${originalFileName}`.toLowerCase();
                const requestBody = {
                    message: `Add image By Trilium: ${fileName}`,
                    content: base64Content
                };

                const { repoPath, token } = this.validatePublishRequirements(note);
                await this.makeGithubRequest(repoPath, token, fileName, requestBody);

                const newImagePath = `/images/${note.noteId}-${originalFileName}`;
                return content.replace(originalText, replacer(newImagePath));
            } catch (error) {
                api.showError(`Failed to process image: ${error.message}`);
            }
        }
        return content;
    }

    async uploadToGithub(note, fileName, content, isBase64 = true) {
        const { repoPath, token } = this.validatePublishRequirements(note);
        const { sha, exists } = await this.checkFileExists(repoPath, token, fileName);

        const requestBody = {
            content: isBase64 ? content : Buffer.from(content).toString('base64'),
            message: exists
                ? `Update by Trilium: ${fileName}`
                : `Create by Trilium: ${fileName}`
        };

        if (sha) {
            requestBody.sha = sha;
        }

        const response = await this.makeGithubRequest(repoPath, token, fileName, requestBody);
        return { ...response, isUpdate: exists };
    }


    async checkFileExists(repoPath, token, fileName) {
        try {
            const response = await fetch(
                `https://api.github.com/repos/${repoPath}/contents/${fileName}`,
                this.getGithubRequestOptions('GET', token)
            );

            if (response.status === 200) {
                const fileData = await response.json();
                return { sha: fileData.sha, exists: true };
            }
            return { sha: null, exists: false };
        } catch (error) {
            console.log('File check failed:', error);
            return { sha: null, exists: false };
        }
    }

    getGithubRequestOptions(method, token, body = null) {
        const options = {
            method,
            headers: {
                'Authorization': `Bearer ${token}`,
                'Accept': 'application/vnd.github.v3+json'
            }
        };

        if (body) {
            options.headers['Content-Type'] = 'application/json';
            options.body = JSON.stringify(body);
        }

        return options;
    }

    async makeGithubRequest(repoPath, token, fileName, body) {
        const response = await fetch(
            `https://api.github.com/repos/${repoPath}/contents/${fileName}`,
            this.getGithubRequestOptions('PUT', token, body)
        );

        if (!response.ok) {
            const errorData = await response.json();
            throw new Error(`GitHub API error: ${errorData.message}`);
        }

        return response.json();
    }

    showPublishResult(result) {
        api.showMessage(result.isUpdate ? 'Note Updated to GitHub!' : 'Note Created to GitHub!');
    }

    processExcerpt(content, frontMatter, note) {
        const excerptAttr = note.getAttribute('label', 'hexoExcerpt');
        if (excerptAttr?.value) {
            frontMatter.excerpt = excerptAttr.value;
            return content;
        }

        const moreSplit = content.split('<!-- more -->');
        if (moreSplit.length > 1) {
            frontMatter.excerpt = moreSplit[0].trim();
        }

        return content;
    }

    async collectFrontMatter(note) {
        const frontMatter = {};
        const attributes = note.getAttributes();

        for (const attr of attributes) {
            if (attr.type === 'label' && attr.name.startsWith('hexo')) {
                let key = attr.name.substring(4);
                key = key.charAt(0).toLowerCase() + key.slice(1);

                if (key === 'tag') {
                    const tagAttrs = note.getAttributes('label', attr.name);
                    frontMatter['tags'] = tagAttrs.map(tagAttr => tagAttr.value);
                } else {
                    frontMatter[key] = attr.value;
                }
            }
        }

        if (!frontMatter.title) {
            frontMatter.title = note.title;
        }

        if (!frontMatter.category) {
            const parentNote = note.getParentNotes()[0];
            frontMatter.category = [parentNote.title];
        }

        const metadata = await note.getMetadata();
        frontMatter.date = metadata.utcDateCreated;
        frontMatter.updated = metadata.utcDateModified;

        return frontMatter;
    }

    generateHexoContent(content, frontMatter) {
        const yaml = Object.entries(frontMatter)
            .map(([key, value]) => {
                if (Array.isArray(value)) {
                    return `${key}:\n${value.map(item => `  - ${item}`).join('\n')}`;
                }
                return `${key}: ${value}`;
            })
            .join('\n');

        return `---\n${yaml}\n---\n\n${content}`;
    }
}

module.exports = new HexoPublisher();