<script>
import $ from 'jquery';
import { mapGetters, mapActions } from 'vuex';
import { escape } from 'underscore';
import TimelineEntryItem from '~/vue_shared/components/notes/timeline_entry_item.vue';
import Flash from '../../flash';
import userAvatarLink from '../../vue_shared/components/user_avatar/user_avatar_link.vue';
import noteHeader from './note_header.vue';
import noteActions from './note_actions.vue';
import noteBody from './note_body.vue';
import eventHub from '../event_hub';
import noteable from '../mixins/noteable';
import resolvable from '../mixins/resolvable';

export default {
  name: 'NoteableNote',
  components: {
    userAvatarLink,
    noteHeader,
    noteActions,
    noteBody,
    TimelineEntryItem,
  },
  mixins: [noteable, resolvable],
  props: {
    note: {
      type: Object,
      required: true,
    },
    line: {
      type: Object,
      required: false,
      default: null,
    },
    helpPagePath: {
      type: String,
      required: false,
      default: '',
    },
  },
  data() {
    return {
      isEditing: false,
      isDeleting: false,
      isRequesting: false,
      isResolving: false,
    };
  },
  computed: {
    ...mapGetters(['targetNoteHash', 'getNoteableData', 'getUserData']),
    author() {
      return this.note.author;
    },
    classNameBindings() {
      return {
        [`note-row-${this.note.id}`]: true,
        'is-editing': this.isEditing && !this.isRequesting,
        'is-requesting being-posted': this.isRequesting,
        'disabled-content': this.isDeleting,
        target: this.isTarget,
        'is-editable': this.note.current_user.can_edit,
      };
    },
    canResolve() {
      return this.note.resolvable && !!this.getUserData.id;
    },
    canReportAsAbuse() {
      return !!this.note.report_abuse_path && this.author.id !== this.getUserData.id;
    },
    noteAnchorId() {
      return `note_${this.note.id}`;
    },
    isTarget() {
      return this.targetNoteHash === this.noteAnchorId;
    },
  },

  created() {
    eventHub.$on('enterEditMode', ({ noteId }) => {
      if (noteId === this.note.id) {
        this.isEditing = true;
        this.scrollToNoteIfNeeded($(this.$el));
      }
    });
  },

  mounted() {
    if (this.isTarget) {
      this.scrollToNoteIfNeeded($(this.$el));
    }
  },

  methods: {
    ...mapActions(['deleteNote', 'updateNote', 'toggleResolveNote', 'scrollToNoteIfNeeded']),
    editHandler() {
      this.isEditing = true;
      this.$emit('handleEdit');
    },
    deleteHandler() {
      const typeOfComment = this.note.isDraft ? 'pending comment' : 'comment';
      // eslint-disable-next-line no-alert
      if (window.confirm(`Are you sure you want to delete this ${typeOfComment}?`)) {
        this.isDeleting = true;
        this.$emit('handleDeleteNote', this.note);

        if (this.note.isDraft) return;

        this.deleteNote(this.note)
          .then(() => {
            this.isDeleting = false;
          })
          .catch(() => {
            Flash('Something went wrong while deleting your note. Please try again.');
            this.isDeleting = false;
          });
      }
    },
    updateSuccess() {
      this.isEditing = false;
      this.isRequesting = false;
      this.oldContent = null;
      $(this.$refs.noteBody.$el).renderGFM();
      this.$refs.noteBody.resetAutoSave();
      this.$emit('updateSuccess');
    },
    formUpdateHandler(noteText, parentElement, callback) {
      this.$emit('handleUpdateNote', {
        note: this.note,
        noteText,
        callback: () => this.updateSuccess(),
      });
      const data = {
        endpoint: this.note.path,
        note: {
          target_type: this.getNoteableData.targetType,
          target_id: this.note.noteable_id,
          note: { note: noteText },
        },
      };
      this.isRequesting = true;
      this.oldContent = this.note.note_html;
      this.note.note_html = escape(noteText);

      this.updateNote(data)
        .then(() => {
          this.updateSuccess();
          callback();
        })
        .catch(() => {
          this.isRequesting = false;
          this.isEditing = true;
          this.$nextTick(() => {
            const msg = 'Something went wrong while editing your comment. Please try again.';
            Flash(msg, 'alert', this.$el);
            this.recoverNoteContent(noteText);
            callback();
          });
        });
    },
    formCancelHandler(shouldConfirm, isDirty) {
      if (shouldConfirm && isDirty) {
        // eslint-disable-next-line no-alert
        if (!window.confirm('Are you sure you want to cancel editing this comment?')) return;
      }
      this.$refs.noteBody.resetAutoSave();
      if (this.oldContent) {
        this.note.note_html = this.oldContent;
        this.oldContent = null;
      }
      this.isEditing = false;
      this.$emit('cancelForm');
    },
    recoverNoteContent(noteText) {
      // we need to do this to prevent noteForm inconsistent content warning
      // this is something we intentionally do so we need to recover the content
      this.note.note = noteText;
      this.$refs.noteBody.note.note = noteText;
    },
  },
};
</script>

<template>
  <timeline-entry-item
    :id="noteAnchorId"
    :class="classNameBindings"
    :data-award-url="note.toggle_award_path"
    :data-note-id="note.id"
    class="note note-wrapper"
  >
    <div v-once class="timeline-icon">
      <user-avatar-link
        :link-href="author.path"
        :img-src="author.avatar_url"
        :img-alt="author.name"
        :img-size="40"
      >
        <slot slot="avatar-badge" name="avatar-badge"></slot>
      </user-avatar-link>
    </div>
    <div class="timeline-content">
      <div class="note-header">
        <note-header
          v-once
          :author="author"
          :created-at="note.created_at"
          :note-id="note.id"
          action-text="commented"
        />
        <note-actions
          :author-id="author.id"
          :note-id="note.id"
          :note-url="note.noteable_note_url"
          :access-level="note.human_access"
          :can-edit="note.current_user.can_edit"
          :can-award-emoji="note.current_user.can_award_emoji"
          :can-delete="note.current_user.can_edit"
          :can-report-as-abuse="canReportAsAbuse"
          :can-resolve="note.current_user.can_resolve"
          :report-abuse-path="note.report_abuse_path"
          :resolvable="note.resolvable"
          :is-resolved="note.resolved"
          :is-resolving="isResolving"
          :resolved-by="note.resolved_by"
          @handleEdit="editHandler"
          @handleDelete="deleteHandler"
          @handleResolve="resolveHandler"
        />
      </div>
      <div class="timeline-discussion-body">
        <slot name="discussion-resolved-text"></slot>
        <note-body
          ref="noteBody"
          :note="note"
          :line="line"
          :can-edit="note.current_user.can_edit"
          :is-editing="isEditing"
          :help-page-path="helpPagePath"
          @handleFormUpdate="formUpdateHandler"
          @cancelForm="formCancelHandler"
        />
      </div>
    </div>
  </timeline-entry-item>
</template>
