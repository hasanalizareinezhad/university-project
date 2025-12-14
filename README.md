<?php
/**
 * Plugin Name: مدیریت و دانلود تکالیف LearnDash + اصلاح موضوع
 * Plugin URI: https://urlnadarad.com
 * Description: افزونه‌ای برای مدیریت، فیلترینگ نیمه‌آبشاری دقیق و دانلود تکالیف دانشجویان LearnDash با رابط کاربری مدرن، تنظیمات دسترسی بر اساس نقش‌های کاربری و اصلاح خودکار.
 * Version: 2
 * Author: Hasanali ZareiNezhad
 * Text Domain: ldash-assignments-manager
 * Domain Path: /languages
 * Requires at least: 5.6
 * Requires PHP: 7.4
 */

/**
 * مقدمه پروژه:
 * این افزونه ابزاری پیشرفته برای مدیریت تکالیف در LearnDash است. ویژگی‌های کلیدی شامل فیلتر نیمه‌آبشاری (دوره → درس → موضوع)، رابط مدرن با آمار، دانلود گروهی ZIP، کنترل دسترسی نقش‌محور، اصلاح متادیتا و پاکسازی خودکار فایل‌ها.
 * نیازمندی‌ها: فیلتر پویا، امنیت بالا، رابط کاربرپسند، سازگاری با LearnDash.
 * اهداف: بهبود کارایی مدیریت تکالیف، امنیت و مقیاس‌پذیری برای مؤسسات آموزشی.
 * نتایج و ارزیابی: کارایی بالا در فیلتر و دانلود، امنیت با nonce و پاکسازی، تجربه کاربری مدرن، سازگاری کامل.
 * جمع‌بندی: افزونه مدیریت تکالیف را تسهیل می‌کند و استانداردهای امنیتی را رعایت می‌نماید.
 */

if (!defined('ABSPATH')) {
    exit; // جلوگیری از دسترسی مستقیم به فایل برای امنیت، استاندارد وردپرس.
}

if (!class_exists('LearnDash_Assignments_Manager')) {
    class LearnDash_Assignments_Manager {
        private static $instance = null; // متغیر استاتیک برای نگهداری نمونه Singleton.
        private $per_page_default = 50; // تعداد پیش‌فرض تکالیف در هر صفحه برای بهینه‌سازی بارگذاری و جلوگیری از بار سنگین سرور.

        /**
         * متد Singleton برای ایجاد تنها یک نمونه از کلاس، جلوگیری از ایجاد چندین نمونه و صرفه‌جویی در منابع.
         * بازگشت: نمونه کلاس.
         */
        public static function get_instance() {
            if (null === self::$instance) {
                self::$instance = new self();
            }
            return self::$instance;
        }

        /**
         * سازنده کلاس: هوک‌های لازم برای منوها، تنظیمات، اسکریپت‌ها، AJAX، اصلاح متادیتا و پاکسازی ZIP را ثبت می‌کند.
         * این متد خصوصی است تا مستقیماً فراخوانی نشود و فقط از طریق Singleton استفاده شود.
         */
        private function __construct() {
            add_action('admin_menu', array($this, 'add_admin_menu')); // هوک برای افزودن منو اصلی.
            add_action('admin_menu', array($this, 'add_settings_submenu')); // هوک برای افزودن زیرمنو تنظیمات.
            add_action('admin_init', array($this, 'register_settings')); // هوک برای ثبت تنظیمات در admin_init.
            add_action('admin_enqueue_scripts', array($this, 'enqueue_scripts')); // هوک برای بارگذاری اسکریپت‌ها و استایل‌ها.

            // هندلرهای AJAX برای بارگذاری داده‌ها و دانلود، همه با پیشوند wp_ajax_.
            add_action('wp_ajax_ldam_get_assignments', array($this, 'ajax_get_assignments')); // هندلر برای بارگذاری تکالیف.
            add_action('wp_ajax_ldam_bulk_download', array($this, 'ajax_bulk_download')); // هندلر برای دانلود گروهی.
            add_action('wp_ajax_ldam_get_lessons_by_course', array($this, 'ajax_get_lessons_by_course')); // هندلر برای درس‌ها.
            add_action('wp_ajax_ldam_get_topics_by_lesson', array($this, 'ajax_get_topics_by_lesson')); // هندلر برای موضوعات.

            // هوک برای اصلاح خودکار lesson_id و topic_id هنگام آپلود تکلیف، با اولویت ۱۰ و ۳ آرگومان.
            add_action('learndash_assignment_process_upload', array($this, 'fix_assignment_topic_metadata'), 10, 3);

            // هوک و برنامه‌ریزی روزانه برای پاکسازی فایل‌های ZIP قدیمی، اگر برنامه‌ریزی نشده باشد.
            add_action('ldam_cleanup_zip_files', array($this, 'cleanup_zip_files'));
            if (!wp_next_scheduled('ldam_cleanup_zip_files')) {
                wp_schedule_event(time(), 'daily', 'ldam_cleanup_zip_files'); // برنامه‌ریزی روزانه اگر وجود نداشته باشد.
            }
        }

        // =========================================================================
        // 1. تنظیمات دسترسی (حفظ شده طبق درخواست)
        // =========================================================================

        /**
         * افزودن زیرمنو برای تنظیمات دسترسی به صفحه اصلی افزونه.
         * دسترسی محدود به مدیران کل با قابلیت manage_options برای امنیت.
         * پارامترها: عنوان صفحه، عنوان منو، قابلیت، slug، تابع رندر.
         */
        public function add_settings_submenu() {
            add_submenu_page(
                'ldash-assignments-manager',
                'تنظیمات دسترسی',
                'تنظیمات دسترسی',
                'manage_options',
                'ldam-access-settings',
                array($this, 'render_access_settings_page')
            );
        }

        /**
         * ثبت تنظیمات نقش‌های مجاز در پایگاه داده وردپرس.
         * از sanitize_callback برای ایمن‌سازی ورودی‌ها استفاده می‌شود تا حملات جلوگیری شود.
         * پارامترها: گروه تنظیمات، نام گزینه، آرایه تنظیمات شامل type، callback و default.
         */
        public function register_settings() {
            register_setting('ldam_access_settings_group', 'ldam_allowed_roles', array(
                'type'              => 'array',
                'sanitize_callback' => array($this, 'sanitize_roles'),
                'default'           => array()
            ));
        }

        /**
         * پاکسازی نقش‌های ورودی با sanitize_key برای امنیت و جلوگیری از ورودی‌های نامعتبر.
         * ورودی: آرایه ورودی.
         * بازگشت: آرایه پاکسازی‌شده یا خالی.
         */
        public function sanitize_roles($input) {
            return is_array($input) ? array_map('sanitize_key', $input) : array();
        }

        /**
         * تولید رابط کاربری صفحه تنظیمات دسترسی.
         * شامل فرم با چک‌باکس‌ها برای نقش‌ها (به جز مدیر کل) و ایمن‌سازی با esc_attr و esc_html.
         * امنیت: بررسی دسترسی مدیر کل و nonce داخلی با settings_fields.
         * دسترسی‌پذیری: استفاده از label و جدول form-table برای صفحه‌خوان‌ها.
         */
        public function render_access_settings_page() {
            if (!current_user_can('manage_options')) {
                wp_die(__('دسترسی غیرمجاز.', 'ldash-assignments-manager'));
            }
            ?>
            <div class="wrap">
                <h1><?php _e('تنظیمات دسترسی به مدیریت تکالیف', 'ldash-assignments-manager'); ?></h1>
                <form action="options.php" method="post">
                    <?php
                    settings_fields('ldam_access_settings_group'); // تولید فیلدهای مخفی شامل nonce.
                    $allowed_roles   = get_option('ldam_allowed_roles', array()); // بازیابی نقش‌های مجاز یا آرایه خالی.
                    $editable_roles  = get_editable_roles(); // دریافت نقش‌های قابل ویرایش.
                    ?>
                    <h2><?php _e('دسترسی‌ها', 'ldash-assignments-manager'); ?></h2>
                    <p><?php _e('نقش‌های کاربری مجاز برای دسترسی به ابزار مدیریت و دانلود تکالیف را انتخاب کنید.', 'ldash-assignments-manager'); ?></p>
                    <p><em><?php _e('مدیران کل همیشه دسترسی دارند.', 'ldash-assignments-manager'); ?></em></p>
                    <table class="form-table">
                        <tr valign="top">
                            <th scope="row"><?php _e('نقش‌های مجاز', 'ldash-assignments-manager'); ?></th>
                            <td>
                                <?php foreach ($editable_roles as $role_key => $role_info) : ?>
                                    <?php if ($role_key === 'administrator') continue; // رد مدیر کل زیرا همیشه مجاز است. ?>
                                    <label>
                                        <input type="checkbox" name="ldam_allowed_roles[]" value="<?php echo esc_attr($role_key); ?>" <?php checked(in_array($role_key, $allowed_roles)); ?>>
                                        <?php echo esc_html(translate_user_role($role_info['name'])); // ترجمه نام نقش برای نمایش. ?>
                                    </label><br>
                                <?php endforeach; ?>
                            </td>
                        </tr>
                    </table>
                    <?php submit_button(__('ذخیره تغییرات', 'ldash-assignments-manager')); // دکمه ذخیره با متن ترجمه‌شده. ?>
                </form>
            </div>
            <?php
        }

        /**
         * بررسی دسترسی کاربر: مدیران کل همیشه مجاز؛ دیگران بر اساس نقش‌های ذخیره‌شده.
         * امنیت: استفاده از array_intersect برای چک نقش‌ها و جلوگیری از دسترسی غیرمجاز.
         * بازگشت: true اگر مجاز، false در غیر این صورت.
         */
        private function has_access() {
            if (current_user_can('manage_options')) {
                return true; // مدیران کل همیشه دسترسی دارند.
            }
            $allowed_roles = get_option('ldam_allowed_roles', array()); // بازیابی نقش‌های مجاز.
            $user          = wp_get_current_user(); // دریافت کاربر فعلی.
            $user_roles    = (array) $user->roles; // تبدیل نقش‌ها به آرایه.
            return !empty(array_intersect($user_roles, $allowed_roles)); // چک تقاطع نقش‌ها.
        }

        // =========================================================================
        // 2. منو و اسکریپت‌ها
        // =========================================================================

        /**
         * افزودن منو اصلی به پیشخوان وردپرس با آیکون و موقعیت مشخص.
         * دسترسی اولیه read برای کاربران واردشده، اما has_access کنترل نهایی را انجام می‌دهد.
         * پارامترها: عنوان صفحه، عنوان منو، قابلیت، slug، تابع رندر، آیکون، موقعیت.
         */
        public function add_admin_menu() {
            add_menu_page(
                'مدیریت تکالیف',
                'تکالیف دانشجویان',
                'read',
                'ldash-assignments-manager',
                array($this, 'render_admin_page'),
                'dashicons-clipboard',
                30
            );
        }

        /**
         * بارگذاری استایل و اسکریپت inline فقط در صفحه افزونه برای بهینه‌سازی.
         * localize_script برای انتقال داده‌های AJAX مانند nonce و ترجمه‌ها به JS.
         * بهینه‌سازی: بررسی هوک و دسترسی برای جلوگیری از بارگذاری غیرضروری در صفحات دیگر.
         * پارامتر: هوک صفحه فعلی.
         */
        public function enqueue_scripts($hook) {
            if ('toplevel_page_ldash-assignments-manager' !== $hook) {
                return; // خروج اگر صفحه افزونه نباشد.
            }
            if (!$this->has_access()) {
                return; // خروج اگر دسترسی نداشته باشد.
            }

            wp_register_style('ldam-admin-style', false); // ثبت استایل inline.
            wp_enqueue_style('ldam-admin-style');
            wp_add_inline_style('ldam-admin-style', $this->get_inline_css()); // افزودن CSS inline.

            wp_register_script('ldam-admin-script', false, array('jquery'), '1.1.7', true); // ثبت اسکریپت inline با وابستگی jQuery.
            wp_enqueue_script('ldam-admin-script');
            wp_add_inline_script('ldam-admin-script', $this->get_inline_js()); // افزودن JS inline.

            wp_localize_script('ldam-admin-script', 'ldamAjax', array( // انتقال داده‌ها به JS.
                'ajaxurl' => admin_url('admin-ajax.php'), // URL AJAX.
                'nonce'   => wp_create_nonce('ldam_nonce'), // nonce برای امنیت.
                'i18n'    => array( // ترجمه‌ها برای JS.
                    'loading'       => __('در حال بارگذاری...', 'ldash-assignments-manager'),
                    'no_assignments'=> __('هیچ تکلیفی یافت نشد', 'ldash-assignments-manager'),
                    'select_one'    => __('لطفاً حداقل یک تکلیف را انتخاب کنید', 'ldash-assignments-manager'),
                )
            ));
        }

        /**
         * تولید CSS inline برای رابط مدرن: گرادینت‌ها، کارت‌ها، جدول responsive، دکمه‌ها و انیمیشن.
         * بهینه‌سازی: inline برای کاهش درخواست‌های HTTP و بارگذاری سریع‌تر.
         * بازگشت: رشته CSS.
         */
        private function get_inline_css() {
            return '
            .ldam-container{max-width:1400px;margin:20px auto;background:#fff;border-radius:8px;box-shadow:0 2px 4px rgba(0,0,0,0.1);} /* ظرف اصلی با حداکثر عرض و سایه. */
            .ldam-header{background:linear-gradient(135deg,#667eea 0%,#764ba2 100%);color:white;padding:30px;border-radius:8px 8px 0 0;} /* هدر با گرادینت. */
            .ldam-header h1{margin:0 0 10px 0;color:white;font-size:28px;} /* عنوان هدر. */
            .ldam-filters{padding:25px;background:#f8f9fa;border-bottom:1px solid #e9ecef;display:grid;grid-template-columns:repeat(auto-fit,minmax(200px,1fr));gap:15px;} /* فیلترها با grid responsive. */
            .ldam-filter-group{display:flex;flex-direction:column;} /* گروه فیلترها. */
            .ldam-filter-group label{font-weight:600;margin-bottom:8px;color:#495057;font-size:13px;} /* لیبل فیلترها. */
            .ldam-filter-group select,.ldam-filter-group input{padding:10px;border:1px solid #ced4da;border-radius:6px;font-size:14px;transition:all .3s;} /* ورودی‌ها با transition. */
            .ldam-filter-group select:focus,.ldam-filter-group input:focus{border-color:#667eea;outline:none;box-shadow:0 0 0 3px rgba(102,126,234,0.1);} /* فوکوس با سایه. */
            .ldam-filter-group select:disabled{background-color:#e9ecef;opacity:0.6;cursor:not-allowed;} /* غیرفعال. */
            .ldam-actions{padding:20px 25px;background:#fff;display:flex;gap:10px;border-bottom:1px solid #e9ecef;} /* اقدامات با flex. */
            .ldam-btn{padding:10px 20px;border:none;border-radius:6px;cursor:pointer;font-size:14px;font-weight:600;transition:all .3s;display:inline-flex;align-items:center;gap:8px;} /* دکمه پایه. */
            .ldam-btn-primary{background:#667eea;color:white;} /* دکمه اصلی. */
            .ldam-btn-primary:hover{background:#5568d3;transform:translateY(-2px);box-shadow:0 4px 8px rgba(102,126,234,0.3);} /* hover اصلی. */
            .ldam-btn-success{background:#10b981;color:white;} /* دکمه موفقیت. */
            .ldam-btn-success:hover{background:#059669;transform:translateY(-2px);box-shadow:0 4px 8px rgba(16,185,129,0.3);} /* hover موفقیت. */
            .ldam-btn-secondary{background:#6c757d;color:white;} /* دکمه ثانویه. */
            .ldam-btn-secondary:hover{background:#5a6268;} /* hover ثانویه. */
            .ldam-content{padding:25px;} /* محتوا. */
            .ldam-stats{display:grid;grid-template-columns:repeat(auto-fit,minmax(200px,1fr));gap:15px;margin-bottom:25px;} /* آمار با grid. */
            .ldam-stat-card{background:linear-gradient(135deg,#f093fb 0%,#f5576c 100%);color:white;padding:20px;border-radius:8px;box-shadow:0 2px 8px rgba(0,0,0,0.1);} /* کارت آمار ۱. */
            .ldam-stat-card:nth-child(2){background:linear-gradient(135deg,#4facfe 0%,#00f2fe 100%);} /* کارت ۲. */
            .ldam-stat-card:nth-child(3){background:linear-gradient(135deg,#43e97b 0%,#38f9d7 100%);} /* کارت ۳. */
            .ldam-stat-card:nth-child(4){background:linear-gradient(135deg,#fa709a 0%,#fee140 100%);} /* کارت ۴. */
            .ldam-stat-label{font-size:13px;opacity:0.9;margin-bottom:5px;} /* لیبل آمار. */
            .ldam-stat-value{font-size:28px;font-weight:bold;} /* مقدار آمار. */
            .ldam-table-wrapper{overflow-x:auto;border-radius:8px;border:1px solid #e9ecef;} /* wrapper جدول با اسکرول. */
            .ldam-table{width:100%;border-collapse:collapse;} /* جدول. */
            .ldam-table thead{background:#f8f9fa;} /* سرجدول. */
            .ldam-table th{padding:15px;text-align:right;font-weight:600;color:#495057;border-bottom:2px solid #dee2e6;font-size:13px;} /* سلول‌های سر. */
            .ldam-table td{padding:15px;border-bottom:1px solid #e9ecef;font-size:14px;} /* سلول‌ها. */
            .ldam-table tbody tr:hover{background:#f8f9fa;} /* hover ردیف‌ها. */
            .ldam-badge{display:inline-block;padding:4px 10px;border-radius:12px;font-size:12px;font-weight:600;} /* بج وضعیت. */
            .ldam-badge-approved{background:#d1fae5;color:#065f46;} /* تأیید شده. */
            .ldam-badge-pending{background:#fef3c7;color:#92400e;} /* در انتظار. */
            .ldam-checkbox{width:18px;height:18px;cursor:pointer;} /* چک‌باکس. */
            .ldam-loading{text-align:center;padding:60px 20px;} /* لودینگ. */
            .ldam-spinner{border:4px solid #f3f4f6;border-top:4px solid #667eea;border-radius:50%;width:50px;height:50px;animation:spin 1s linear infinite;margin:0 auto 20px;} /* spinner. */
            @keyframes spin{0%{transform:rotate(0deg);}100%{transform:rotate(360deg);}} /* انیمیشن چرخش. */
            .ldam-empty{text-align:center;padding:60px 20px;color:#6c757d;} /* خالی. */
            .ldam-empty-icon{font-size:64px;margin-bottom:20px;opacity:0.5;} /* آیکون خالی. */
            .ldam-action-btn{padding:6px 12px;margin:0 3px;border:none;border-radius:4px;cursor:pointer;font-size:12px;transition:all .2s;} /* دکمه عملیات. */
            .ldam-download-btn{background:#10b981;color:white;} /* دانلود. */
            .ldam-download-btn:hover{background:#059669;} /* hover دانلود. */
            .ldam-view-btn{background:#3b82f6;color:white;} /* مشاهده. */
            .ldam-view-btn:hover{background:#2563eb;} /* hover مشاهده. */
            ';
        }

        /**
         * تولید JS inline برای مدیریت پویا: فیلترها، AJAX، دانلود گروهی، رندر جدول و آمار.
         * امنیت: esc_html برای خروجی، nonce در درخواست‌ها، جلوگیری از XSS.
         * بهینه‌سازی: جلوگیری از لودینگ همزمان با isLoading، ذخیره فیلترها در currentFilters، استفاده از strict mode.
         * بازگشت: رشته JS.
         */
        private function get_inline_js() {
            return '
            jQuery(document).ready(function($) {
                "use strict"; // فعال‌سازی strict mode برای جلوگیری از خطاهای JS.
                let currentFilters = { course: "", lesson: "", topic: "", user: "" }; // ذخیره فیلترهای فعلی برای مدیریت وابستگی.
                let per_page = ' . intval($this->per_page_default) . '; // تعداد تکالیف در هر صفحه، همخوانی با سمت سرور.
                let isLoading = false; // فلگ برای جلوگیری از درخواست‌های همزمان AJAX.

                function esc_html(s) { // تابع برای ایمن‌سازی خروجی HTML و جلوگیری از XSS در JS.
                    return String(s).replace(/[&<>"\']]/g, function(m){
                        return {"&":"&amp;","<":"&lt;",">":"&gt;","\"":"&quot;","\'":"&#039;"}[m];
                    });
                }

                function updateFilterStates() { // به‌روزرسانی وضعیت غیرفعال/فعال فیلترهای وابسته (cascading).
                    var courseSelected = $("#filter-course").val() !== ""; // چک انتخاب دوره.
                    var lessonSelected = $("#filter-lesson").val() !== ""; // چک انتخاب درس.
                    $("#filter-lesson").prop("disabled", !courseSelected); // غیرفعال اگر دوره انتخاب نشده.
                    if (!courseSelected) {
                        $("#filter-lesson").html("<option value=\"\">همه درس‌ها</option>"); // ریست درس‌ها.
                        currentFilters.lesson = ""; // ریست فیلتر درس.
                    }
                    $("#filter-topic").prop("disabled", !lessonSelected); // غیرفعال اگر درس انتخاب نشده.
                    if (!lessonSelected) {
                        $("#filter-topic").html("<option value=\"\">همه موضوعات</option>"); // ریست موضوعات.
                        currentFilters.topic = ""; // ریست فیلتر موضوع.
                    }
                }

                $("#filter-course").on("change", function() { // رویداد تغییر دوره: بارگذاری درس‌ها با AJAX.
                    var courseId = $(this).val(); // دریافت ID دوره.
                    currentFilters.course = courseId; // ذخیره فیلتر.
                    currentFilters.lesson = ""; // ریست درس.
                    currentFilters.topic = ""; // ریست موضوع.
                    if (courseId) { // اگر دوره انتخاب شده.
                        $("#filter-lesson").html("<option value=\"\">در حال بارگذاری...</option>").prop("disabled", false); // لودینگ درس.
                        $("#filter-topic").html("<option value=\"\">ابتدا درس را انتخاب کنید</option>").prop("disabled", true); // غیرفعال موضوع.
                        $.ajax({ // درخواست AJAX برای درس‌ها.
                            url: ldamAjax.ajaxurl,
                            method: "POST",
                            dataType: "json",
                            data: { action: "ldam_get_lessons_by_course", nonce: ldamAjax.nonce, course_id: courseId },
                            success: function(res) { // موفقیت: تولید گزینه‌ها.
                                var options = "<option value=\"\">همه درس‌ها</option>";
                                if (res.success && res.data && res.data.length) { // اگر داده وجود داشته باشد.
                                    res.data.forEach(function(lesson) {
                                        options += "<option value=\"" + lesson.id + "\">" + esc_html(lesson.title) + "</option>"; // افزودن گزینه با esc_html.
                                    });
                                } else {
                                    options = "<option value=\"\" disabled>هیچ درسی یافت نشد</option>"; // بدون درس.
                                }
                                $("#filter-lesson").html(options); // به‌روزرسانی select.
                                updateFilterStates(); // به‌روزرسانی وضعیت.
                                loadAssignments(); // بارگذاری تکالیف جدید.
                            }
                        });
                    } else { // اگر دوره انتخاب نشده.
                        $("#filter-lesson").html("<option value=\"\">همه درس‌ها</option>"); // ریست.
                        $("#filter-topic").html("<option value=\"\">همه موضوعات</option>"); // ریست.
                        updateFilterStates(); // به‌روزرسانی.
                        loadAssignments(); // بارگذاری.
                    }
                });

                $("#filter-lesson").on("change", function() { // رویداد تغییر درس: بارگذاری موضوعات با AJAX.
                    var lessonId = $(this).val(); // دریافت ID درس.
                    currentFilters.lesson = lessonId; // ذخیره فیلتر.
                    currentFilters.topic = ""; // ریست موضوع.
                    if (lessonId) { // اگر درس انتخاب شده.
                        $("#filter-topic").html("<option value=\"\">در حال بارگذاری...</option>").prop("disabled", false); // لودینگ موضوع.
                        $.ajax({ // درخواست AJAX برای موضوعات.
                            url: ldamAjax.ajaxurl,
                            method: "POST",
                            dataType: "json",
                            data: { action: "ldam_get_topics_by_lesson", nonce: ldamAjax.nonce, lesson_id: lessonId, course_id: currentFilters.course },
                            success: function(res) { // موفقیت: تولید گزینه‌ها.
                                var options = "<option value=\"\">همه موضوعات</option>";
                                if (res.success && res.data && res.data.length) { // اگر داده وجود داشته باشد.
                                    res.data.forEach(function(topic) {
                                        options += "<option value=\"" + topic.id + "\">" + esc_html(topic.title) + "</option>"; // افزودن با esc_html.
                                    });
                                } else {
                                    options += "<option value=\"\" disabled>هیچ موضوعی یافت نشد</option>"; // بدون موضوع.
                                }
                                $("#filter-topic").html(options); // به‌روزرسانی select.
                                updateFilterStates(); // به‌روزرسانی وضعیت.
                                loadAssignments(); // بارگذاری تکالیف.
                            }
                        });
                    } else { // اگر درس انتخاب نشده.
                        $("#filter-topic").html("<option value=\"\">همه موضوعات</option>"); // ریست.
                        updateFilterStates(); // به‌روزرسانی.
                        loadAssignments(); // بارگذاری.
                    }
                });

                $("#filter-topic, #filter-user").on("change", function() { // رویداد تغییر موضوع یا دانشجو: بارگذاری مستقیم.
                    var key = $(this).attr("id").replace("filter-", ""); // استخراج کلید فیلتر.
                    currentFilters[key] = $(this).val(); // ذخیره فیلتر.
                    loadAssignments(); // بارگذاری تکالیف.
                });

                $("#btn-bulk-download").on("click", function() { // رویداد دانلود گروهی: جمع‌آوری IDها و AJAX.
                    var $btn = $(this); // ذخیره دکمه برای تغییر متن.
                    var selectedIds = []; // آرایه IDهای انتخاب‌شده.
                    $(".assignment-checkbox:checked").each(function() { // جمع‌آوری چک‌شده‌ها.
                        selectedIds.push($(this).data("id"));
                    });
                    if (selectedIds.length === 0) { // اگر هیچ انتخاب نشده.
                        alert(ldamAjax.i18n.select_one); // هشدار.
                        return;
                    }
                    var originalText = $btn.html(); // ذخیره متن اصلی.
                    $btn.html("در حال ساخت...").prop("disabled", true); // تغییر به لودینگ.
                    $.ajax({ // درخواست AJAX برای دانلود.
                        url: ldamAjax.ajaxurl,
                        method: "POST",
                        dataType: "json",
                        data: { action: "ldam_bulk_download", nonce: ldamAjax.nonce, ids: selectedIds.join(",") },
                        success: function(res) { // موفقیت: دانلود فایل.
                            if (res.success && res.data.url) {
                                var link = document.createElement("a"); // ایجاد لینک دانلود.
                                link.href = res.data.url;
                                link.setAttribute("download", "Learndash_Assignments_Bulk.zip");
                                document.body.appendChild(link);
                                link.click(); // کلیک خودکار.
                                document.body.removeChild(link); // حذف لینک.
                            } else {
                                var msg = res.data && res.data.message ? res.data.message : "خطای ناشناخته در سرور رخ داد."; // پیام خطا.
                                alert(msg);
                            }
                        },
                        error: function() { // خطا: هشدار ارتباط.
                            alert("خطا در ارتباط با سرور. ممکن است فایل‌ها حجیم بوده یا زمان پردازش تمام شده باشد.");
                        },
                        complete: function() { // پایان: بازگردانی دکمه.
                            $btn.html(originalText).prop("disabled", false);
                        }
                    });
                });

                $("#btn-reset").on("click", function() { // رویداد بازنشانی: ریست همه فیلترها و بارگذاری.
                    currentFilters = { course: "", lesson: "", topic: "", user: "" }; // ریست فیلترها.
                    $("#filter-course, #filter-lesson, #filter-topic, #filter-user").val(""); // ریست selectها.
                    $("#filter-lesson").html("<option value=\"\">همه درس‌ها</option>"); // ریست درس.
                    $("#filter-topic").html("<option value=\"\">همه موضوعات</option>"); // ریست موضوع.
                    updateFilterStates(); // به‌روزرسانی وضعیت.
                    loadAssignments(); // بارگذاری تکالیف.
                });

                function loadAssignments(page = 1) { // تابع بارگذاری تکالیف با AJAX و pagination (page پیش‌فرض ۱).
                    if (isLoading) return; // جلوگیری از درخواست همزمان.
                    isLoading = true; // تنظیم فلگ.
                    $("#ldam-assignments-table").html("<div class=\"ldam-loading\"><div class=\"ldam-spinner\"></div><p>" + ldamAjax.i18n.loading + "</p></div>"); // نمایش لودینگ.
                    $.ajax({ // درخواست AJAX.
                        url: ldamAjax.ajaxurl,
                        type: "POST",
                        dataType: "json",
                        data: { action: "ldam_get_assignments", nonce: ldamAjax.nonce, filters: currentFilters, per_page: per_page, paged: page },
                        success: function(response) { // موفقیت: رندر داده‌ها.
                            isLoading = false; // ریست فلگ.
                            if (response.success) {
                                renderAssignments(response.data); // رندر تکالیف.
                            } else {
                                var msg = response.data && response.data.message ? response.data.message : "خطا در بارگذاری تکالیف"; // پیام خطا.
                                $("#ldam-assignments-table").html("<div class=\"ldam-empty\"><div class=\"ldam-empty-icon\">(clipboard)</div><p>" + msg + "</p></div>"); // نمایش خالی.
                                updateStats({total:0,approved:0,pending:0,rejected:0}); // ریست آمار.
                            }
                        },
                        error: function() { // خطا: نمایش پیام.
                            isLoading = false;
                            $("#ldam-assignments-table").html("<div class=\"ldam-empty\"><div class=\"ldam-empty-icon\">(warning)</div><p>خطا در ارتباط با سرور</p></div>");
                        }
                    });
                }

                function renderAssignments(data) { // تابع رندر تکالیف: تولید جدول و به‌روزرسانی آمار.
                    if (!data.assignments || data.assignments.length === 0) { // اگر خالی.
                        $("#ldam-assignments-table").html("<div class=\"ldam-empty\"><div class=\"ldam-empty-icon\">(clipboard)</div><h3>" + ldamAjax.i18n.no_assignments + "</h3><p>با استفاده از فیلترها می‌توانید جستجو کنید</p></div>"); // نمایش خالی.
                        updateStats(data.stats || {total:0,approved:0,pending:0,rejected:0}); // ریست آمار.
                        return;
                    }
                    updateStats(data.stats); // به‌روزرسانی آمار.
                    var html = "<div class=\"ldam-table-wrapper\"><table class=\"ldam-table\"><thead><tr>"; // شروع جدول.
                    html += "<th><input type=\"checkbox\" id=\"select-all\" class=\"ldam-checkbox\"></th>"; // چک همه.
                    html += "<th>دانشجو</th><th>دوره</th><th>درس</th><th>موضوع</th><th>تاریخ ارسال</th><th>وضعیت</th><th>عملیات</th>"; // سرستون‌ها.
                    html += "</tr></thead><tbody>"; // شروع بدنه.
                    data.assignments.forEach(function(assignment) { // حلقه روی تکالیف.
                        var statusClass = assignment.status === "approved" ? "approved" : (assignment.status === "not_approved" ? "not-approved" : "pending"); // کلاس وضعیت.
                        var statusText = assignment.status === "approved" ? "تأیید شده" : (assignment.status === "not_approved" ? "رد شده" : "در انتظار"); // متن وضعیت.
                        html += "<tr>"; // شروع ردیف.
                        html += "<td><input type=\"checkbox\" class=\"ldam-checkbox assignment-checkbox\" data-id=\"" + assignment.id + "\"></td>"; // چک‌باکس.
                        html += "<td>" + esc_html(assignment.user_name) + "</td>"; // دانشجو.
                        html += "<td>" + esc_html(assignment.course_title) + "</td>"; // دوره.
                        html += "<td>" + esc_html(assignment.lesson_title) + "</td>"; // درس.
                        html += "<td>" + esc_html(assignment.topic_title) + "</td>"; // موضوع.
                        html += "<td>" + esc_html(assignment.date) + "</td>"; // تاریخ.
                        html += "<td><span class=\"ldam-badge ldam-badge-" + statusClass + "\">" + statusText + "</span></td>"; // وضعیت.
                        html += "<td>"; // عملیات.
                        if (assignment.file_url) { // اگر URL فایل وجود داشته باشد.
                            html += "<a class=\"ldam-action-btn ldam-download-btn\" href=\"" + assignment.file_url + "\" download>دانلود</a>"; // لینک دانلود.
                        }
                        html += "<button class=\"ldam-action-btn ldam-view-btn\" data-url=\"" + assignment.view_url + "\">مشاهده</button>"; // دکمه مشاهده.
                        html += "</td></tr>"; // پایان ردیف.
                    });
                    html += "</tbody></table></div>"; // پایان جدول.
                    $("#ldam-assignments-table").html(html); // به‌روزرسانی DOM.
                    $("#select-all").off("change").on("change", function() { // رویداد چک همه.
                        $(".assignment-checkbox").prop("checked", $(this).prop("checked")); // چک/آنچک همه.
                    });
                    $(".ldam-view-btn").off("click").on("click", function(){ // رویداد مشاهده.
                        window.open($(this).data("url"), "_blank", "noopener"); // باز کردن در تب جدید.
                    });
                }

                function updateStats(stats) { // تابع به‌روزرسانی کارت‌های آمار.
                    stats = stats || {total:0,approved:0,pending:0,rejected:0}; // پیش‌فرض اگر null.
                    $("#stat-total").text(stats.total); // کل.
                    $("#stat-approved").text(stats.approved); // تأیید شده.
                    $("#stat-pending").text(stats.pending); // در انتظار.
                    $("#stat-rejected").text(stats.rejected); // رد شده.
                }

                updateFilterStates(); // فراخوانی اولیه برای وضعیت فیلترها.
                loadAssignments(); // بارگذاری اولیه تکالیف.
            });
            ';
        }

        /**
         * تولید رابط کاربری صفحه اصلی: هدر، فیلترها، اقدامات، آمار و جدول.
         * امنیت: بررسی دسترسی و esc_html برای خروجی‌ها.
         * دسترسی‌پذیری: labelها و ساختار grid برای فیلترها، جدول با scope.
         * بهینه‌سازی: تولید گزینه‌ها با متدهای جداگانه.
         */
        public function render_admin_page() {
            if (!$this->has_access()) {
                wp_die(__('شما برای دسترسی به این صفحه دسترسی کافی ندارید.', 'ldash-assignments-manager')); // خروج اگر غیرمجاز.
            }
            ?>
            <div class="wrap"> <!-- ظرف اصلی پیشخوان. -->
                <div class="ldam-container"> <!-- ظرف افزونه با استایل. -->
                    <div class="ldam-header"> <!-- هدر. -->
                        <h1>مدیریت تکالیف دانشجویان</h1>
                        <p>مدیریت و دانلود تکالیف ارسالی دانشجویان LearnDash</p>
                    </div>
                    <div class="ldam-filters"> <!-- فیلترها. -->
                        <div class="ldam-filter-group"> <!-- گروه دوره. -->
                            <label>دوره:</label>
                            <select id="filter-course">
                                <option value=""><?php echo esc_html__('همه دوره‌ها','ldash-assignments-manager'); ?></option>
                                <?php echo $this->get_courses_options(); // گزینه‌های دوره. ?>
                            </select>
                        </div>
                        <div class="ldam-filter-group"> <!-- گروه درس. -->
                            <label>درس:</label>
                            <select id="filter-lesson">
                                <option value=""><?php echo esc_html__('همه درس‌ها','ldash-assignments-manager'); ?></option>
                            </select>
                        </div>
                        <div class="ldam-filter-group"> <!-- گروه موضوع. -->
                            <label>موضوع:</label>
                            <select id="filter-topic">
                                <option value=""><?php echo esc_html__('همه موضوعات','ldash-assignments-manager'); ?></option>
                            </select>
                        </div>
                        <div class="ldam-filter-group"> <!-- گروه دانشجو. -->
                            <label>دانشجو:</label>
                            <select id="filter-user">
                                <option value=""><?php echo esc_html__('همه دانشجویان','ldash-assignments-manager'); ?></option>
                                <?php echo $this->get_users_options(); // گزینه‌های دانشجو. ?>
                            </select>
                        </div>
                    </div>
                    <div class="ldam-actions"> <!-- اقدامات. -->
                        <button id="btn-bulk-download" class="ldam-btn ldam-btn-success">دانلود گروهی</button>
                        <button id="btn-reset" class="ldam-btn ldam-btn-secondary">بازنشانی</button>
                    </div>
                    <div class="ldam-content"> <!-- محتوا. -->
                        <div class="ldam-stats"> <!-- آمار. -->
                            <div class="ldam-stat-card"><div class="ldam-stat-label"><?php _e('کل تکالیف','ldash-assignments-manager'); ?></div><div class="ldam-stat-value" id="stat-total">0</div></div>
                            <div class="ldam-stat-card"><div class="ldam-stat-label"><?php _e('تأیید شده','ldash-assignments-manager'); ?></div><div class="ldam-stat-value" id="stat-approved">0</div></div>
                            <div class="ldam-stat-card"><div class="ldam-stat-label"><?php _e('در انتظار','ldash-assignments-manager'); ?></div><div class="ldam-stat-value" id="stat-pending">0</div></div>
                            <div class="ldam-stat-card"><div class="ldam-stat-label"><?php _e('رد شده','ldash-assignments-manager'); ?></div><div class="ldam-stat-value" id="stat-rejected">0</div></div>
                        </div>
                        <div id="ldam-assignments-table"> <!-- جدول تکالیف. -->
                            <div class="ldam-loading"><div class="ldam-spinner"></div><p><?php _e('در حال بارگذاری تکالیف...','ldash-assignments-manager'); ?></p></div>
                        </div>
                    </div>
                </div>
            </div>
            <?php
        }

        /**
         * تولید گزینه‌های select برای دوره‌ها با مرتب‌سازی الفبایی بر اساس عنوان.
         * امنیت: esc_attr برای value و esc_html برای متن.
         * بهینه‌سازی: posts_per_page=-1 برای دریافت همه، اما در سایت‌های بزرگ ممکن است سنگین باشد.
         * بازگشت: رشته HTML گزینه‌ها.
         */
        private function get_courses_options() {
            $courses = get_posts(array('post_type' => 'sfwd-courses', 'posts_per_page' => -1, 'orderby' => 'title', 'order' => 'ASC')); // دریافت دوره‌ها.
            $options = ''; // رشته گزینه‌ها.
            foreach ($courses as $course) { // حلقه روی دوره‌ها.
                $options .= '<option value="' . esc_attr($course->ID) . '">' . esc_html($course->post_title) . '</option>'; // افزودن گزینه.
            }
            return $options;
        }

        /**
         * تولید گزینه‌های select برای دانشجویان (حداکثر ۵۰۰ برای جلوگیری از بار سنگین) با مرتب‌سازی بر اساس نام نمایشی.
         * امنیت: esc_attr برای value و esc_html برای متن.
         * بهینه‌سازی: number=500 برای محدودیت، در سایت‌های بزرگ ممکن است نیاز به pagination باشد.
         * بازگشت: رشته HTML گزینه‌ها.
         */
        private function get_users_options() {
            $users = get_users(array('orderby' => 'display_name', 'number' => 500)); // دریافت کاربران.
            $options = ''; // رشته گزینه‌ها.
            foreach ($users as $user) { // حلقه روی کاربران.
                $options .= '<option value="' . esc_attr($user->ID) . '">' . esc_html($user->display_name) . '</option>'; // افزودن گزینه.
            }
            return $options;
        }

        // AJAX Handlers

        /**
         * AJAX برای بارگذاری درس‌های یک دوره.
         * امنیت: check_ajax_referer برای CSRF و has_access برای دسترسی؛ intval برای course_id.
         * fallback به get_posts اگر تابع learndash_get_course_lessons_list موجود نباشد.
         * خروجی: آرایه JSON با id و title درس‌ها.
         */
        public function ajax_get_lessons_by_course() {
            check_ajax_referer('ldam_nonce', 'nonce'); // بررسی nonce برای امنیت.
            if (!$this->has_access()) wp_send_json_error('دسترسی غیرمجاز'); // بررسی دسترسی.
            $course_id = isset($_POST['course_id']) ? intval($_POST['course_id']) : 0; // پاکسازی ورودی.
            if (!$course_id) wp_send_json_success(array()); // بازگشت خالی اگر ID صفر.

            $data = array(); // آرایه داده‌ها.
            if (function_exists('learndash_get_course_lessons_list')) { // اگر تابع LearnDash موجود باشد.
                $lessons = learndash_get_course_lessons_list($course_id); // دریافت لیست درس‌ها.
                foreach ($lessons as $lesson) { // حلقه.
                    if (isset($lesson['post']) && $lesson['post'] instanceof WP_Post) { // چک پست.
                        $data[] = array('id' => $lesson['post']->ID, 'title' => $lesson['post']->post_title); // افزودن.
                    }
                }
            } else { // fallback.
                $posts = get_posts(array(
                    'post_type'      => 'sfwd-lessons',
                    'posts_per_page' => -1,
                    'meta_key'       => 'course_id',
                    'meta_value'     => $course_id,
                    'orderby'        => 'title',
                    'order'          => 'ASC'
                )); // دریافت با query.
                foreach ($posts as $p) $data[] = array('id' => $p->ID, 'title' => $p->post_title); // افزودن.
            }
            wp_send_json_success($data); // ارسال موفقیت.
        }

        /**
         * AJAX برای بارگذاری موضوعات یک درس.
         * امنیت: check_ajax_referer برای CSRF و has_access برای دسترسی؛ intval برای ورودی‌ها.
         * fallback به get_posts با meta_query دوگانه (lesson_id یا post_parent) اگر تابع LearnDash موجود نباشد.
         * فیلتر بر اساس course_id، حذف تکراری‌ها با $seen، مرتب‌سازی الفبایی با usort.
         * خروجی: آرایه JSON منحصربه‌فرد با id و title موضوعات.
         */
        public function ajax_get_topics_by_lesson() {
            check_ajax_referer('ldam_nonce', 'nonce'); // بررسی nonce.
            if (!$this->has_access()) {
                wp_send_json_error('دسترسی غیرمجاز'); // بررسی دسترسی.
            }

            $lesson_id = isset($_POST['lesson_id']) ? intval($_POST['lesson_id']) : 0; // پاکسازی درس.
            $course_id = isset($_POST['course_id']) ? intval($_POST['course_id']) : 0; // پاکسازی دوره.

            if (!$lesson_id) {
                wp_send_json_success(array()); // بازگشت خالی اگر درس صفر.
            }

            $data = array(); // آرایه داده‌ها.

            if (function_exists('learndash_get_topic_list')) { // اگر تابع LearnDash موجود باشد.
                $topics = learndash_get_topic_list($lesson_id, $course_id); // دریافت لیست.
                if (!empty($topics)) {
                    foreach ($topics as $topic) { // حلقه.
                        if ($topic instanceof WP_Post && $topic->post_type === 'sfwd-topic') { // چک نوع.
                            $data[] = array(
                                'id'    => $topic->ID,
                                'title' => $topic->post_title
                            ); // افزودن.
                        }
                    }
                }
            }

            if (empty($data)) { // fallback.
                $topics = get_posts(array(
                    'post_type'      => 'sfwd-topic',
                    'posts_per_page' => -1,
                    'post_status'    => 'publish',
                    'orderby'        => 'title',
                    'order'          => 'ASC',
                    'meta_query'     => array(
                        'relation' => 'OR', // OR برای lesson_id یا parent.
                        array(
                            'key'     => 'lesson_id',
                            'value'   => $lesson_id,
                            'compare' => '='
                        ),
                        array(
                            'key'     => 'post_parent',
                            'value'   => $lesson_id,
                            'compare' => '='
                        )
                    )
                )); // دریافت با query.

                foreach ($topics as $topic) { // حلقه.
                    if ($course_id) { // فیلتر دوره اگر مشخص.
                        $topic_course_id = get_post_meta($topic->ID, 'course_id', true);
                        if ($topic_course_id && $topic_course_id != $course_id) {
                            continue; // رد اگر دوره مطابقت نکند.
                        }
                    }
                    $data[] = array(
                        'id'    => $topic->ID,
                        'title' => $topic->post_title
                    ); // افزودن.
                }
            }

            $unique = array(); // آرایه منحصربه‌فرد.
            $seen   = array(); // دیده‌شده‌ها برای حذف تکراری.
            foreach ($data as $item) {
                if (!in_array($item['id'], $seen)) { // چک تکراری.
                    $unique[] = $item;
                    $seen[]   = $item['id'];
                }
            }

            usort($unique, function($a, $b) { // مرتب‌سازی الفبایی بر اساس عنوان.
                return strcmp($a['title'], $b['title']);
            });

            wp_send_json_success($unique); // ارسال موفقیت.
        }

        /**
         * AJAX برای بارگذاری تکالیف بر اساس فیلترها با WP_Query.
         * امنیت: check_ajax_referer برای CSRF و has_access برای دسترسی؛ intval برای ورودی‌ها و فیلترها.
         * محاسبه آمار وضعیت (total, approved, pending, rejected)؛ استخراج متادیتا، URL دانلود و مشاهده.
         * fallback برای عناوین نامشخص اگر پست وجود نداشته باشد.
         * بهینه‌سازی: محدودیت per_page تا ۲۰۰ برای جلوگیری از بار سنگین؛ مرتب‌سازی بر اساس تاریخ DESC.
         * خروجی: JSON با assignments (آرایه داده‌ها) و stats (آمار).
         */
        public function ajax_get_assignments() {
            check_ajax_referer('ldam_nonce', 'nonce'); // بررسی nonce.
            if (!$this->has_access()) wp_send_json_error(array('message' => __('دسترسی غیرمجاز','ldash-assignments-manager'))); // بررسی دسترسی.

            $filters  = isset($_POST['filters']) && is_array($_POST['filters']) ? $_POST['filters'] : array(); // دریافت فیلترها.
            $paged    = isset($_POST['paged']) ? max(1, intval($_POST['paged'])) : 1; // صفحه، حداقل ۱.
            $per_page = isset($_POST['per_page']) ? max(1, min(200, intval($_POST['per_page']))) : $this->per_page_default; // تعداد، بین ۱-۲۰۰.

            $args = array( // آرایه query پایه.
                'post_type'      => 'sfwd-assignment',
                'posts_per_page' => $per_page,
                'paged'          => $paged,
                'orderby'        => 'date',
                'order'          => 'DESC'
            );

            $meta_query = array('relation' => 'AND'); // meta_query با AND.
            if (!empty($filters['course']))  $meta_query[] = array('key' => 'course_id',  'value' => intval($filters['course']),  'compare' => '='); // فیلتر دوره.
            if (!empty($filters['lesson']))  $meta_query[] = array('key' => 'lesson_id',  'value' => intval($filters['lesson']),  'compare' => '='); // فیلتر درس.
            if (!empty($filters['topic']))   $meta_query[] = array('key' => 'topic_id',   'value' => intval($filters['topic']),   'compare' => '='); // فیلتر موضوع.
            if (!empty($filters['user']))    $args['author'] = intval($filters['user']); // فیلتر دانشجو.

            if (count($meta_query) > 1) $args['meta_query'] = $meta_query; // افزودن اگر بیش از ۱.

            $query = new WP_Query($args); // اجرای query.
            $assignments = array(); // آرایه تکالیف.
            $stats = array('total' => 0, 'approved' => 0, 'pending' => 0, 'rejected' => 0); // آمار اولیه.

            if ($query->have_posts()) { // اگر پست وجود داشته باشد.
                while ($query->have_posts()) { // حلقه.
                    $query->the_post(); // تنظیم پست.
                    $post_id = get_the_ID(); // ID پست.
                    if ('sfwd-assignment' !== get_post_type($post_id)) continue; // رد اگر نوع اشتباه.

                    $approval_status = get_post_meta($post_id, 'approval_status', true); // وضعیت تأیید.
                    $file_url        = function_exists('learndash_assignment_get_download_url') ? learndash_assignment_get_download_url($post_id) : ''; // URL دانلود.
                    $user_obj        = get_userdata(get_the_author_meta('ID')); // کاربر.

                    $course = $lesson = $topic = null; // متغیرها.
                    $course_id = get_post_meta($post_id, 'course_id', true); // متا دوره.
                    $lesson_id = get_post_meta($post_id, 'lesson_id', true); // متا درس.
                    $topic_id  = get_post_meta($post_id, 'topic_id', true); // متا موضوع.
                    if ($course_id) $course = get_post($course_id); // دریافت پست دوره.
                    if ($lesson_id) $lesson = get_post($lesson_id); // درس.
                    if ($topic_id)  $topic  = get_post($topic_id); // موضوع.

                    $assignments[] = array( // افزودن به آرایه.
                        'id'           => $post_id,
                        'user_name'    => $user_obj ? $user_obj->display_name : __('نامشخص','ldash-assignments-manager'),
                        'course_title' => $course ? $course->post_title : __('نامشخص','ldash-assignments-manager'),
                        'lesson_title' => $lesson ? $lesson->post_title : __('نامشخص','ldash-assignments-manager'),
                        'topic_title'  => $topic  ? $topic->post_title  : 'بدون موضوع',
                        'date'         => get_the_date('Y/m/d H:i', $post_id), // تاریخ فرمت‌شده.
                        'status'       => $approval_status ?: 'pending', // پیش‌فرض pending.
                        'file_url'     => $file_url,
                        'view_url'     => esc_url_raw(get_edit_post_link($post_id)) // URL ویرایش.
                    );

                    $stats['total']++; // افزایش کل.
                    if ($approval_status === 'approved') $stats['approved']++; // تأیید.
                    elseif ($approval_status === 'not_approved') $stats['rejected']++; // رد.
                    else $stats['pending']++; // انتظار.
                }
                wp_reset_postdata(); // ریست query.
            }

            wp_send_json_success(array('assignments' => $assignments, 'stats' => $stats)); // ارسال موفقیت.
        }

        /**
         * AJAX برای دانلود گروهی به ZIP با نام‌گذاری هوشمند (دانشجو_دوره_درس_موضوع_ID.ext).
         * امنیت: check_ajax_referer برای CSRF و has_access برای دسترسی؛ sanitize_text_field و intval برای ids.
         * بهینه‌سازی: افزایش time_limit به ۶۰۰ ثانیه و memory_limit به ۱GB برای فایل‌های حجیم.
         * fallback برای یافتن مسیر فایل اگر متا ناقص باشد (چک چندین مسیر ممکن).
         * افزودن comment به ZIP برای شناسایی؛ پاکسازی خودکار پس از ساخت با cleanup_zip_files.
         * مدیریت خطاها: فایل گم‌شده، عدم ساخت ZIP، با جمع‌آوری errors و ارسال پیام.
         * خروجی: URL ZIP اگر موفق، иначе پیام خطا.
         */
        public function ajax_bulk_download() {
            check_ajax_referer('ldam_nonce', 'nonce'); // بررسی nonce.
            if (!$this->has_access()) wp_send_json_error(array('message' => __('دسترسی غیرمجاز.', 'ldash-assignments-manager'))); // بررسی دسترسی.

            $ids_raw = isset($_POST['ids']) ? sanitize_text_field($_POST['ids']) : ''; // دریافت خام ids.
            if (empty($ids_raw)) wp_send_json_error(array('message' => __('هیچ تکلیفی انتخاب نشده است.', 'ldash-assignments-manager'))); // چک خالی.

            $ids = array_filter(array_map('intval', explode(',', $ids_raw))); // پاکسازی به آرایه intval.
            if (empty($ids)) wp_send_json_error(array('message' => __('شناسه‌های نامعتبر.', 'ldash-assignments-manager'))); // چک معتبر.
            if (!class_exists('ZipArchive')) wp_send_json_error(array('message' => __('کلاس ZipArchive روی سرور شما فعال نیست.', 'ldash-assignments-manager'))); // چک کلاس ZIP.

            @set_time_limit(600); // افزایش زمان اجرا.
            @ini_set('memory_limit', '1024M'); // افزایش حافظه.

            $upload_dir   = wp_upload_dir(); // دایرکتوری آپلود.
            $zip_dir_path = $upload_dir['basedir'] . '/ldam-zips'; // مسیر ZIPها.
            $zip_dir_url  = $upload_dir['baseurl'] . '/ldam-zips'; // URL.

            if (!is_dir($zip_dir_path)) wp_mkdir_p($zip_dir_path); // ایجاد دایرکتوری اگر不存在.

            $zip_file_name = 'Assignments_' . date('Y-m-d_H-i-s') . '_' . wp_generate_password(8, false) . '.zip'; // نام منحصربه‌فرد.
            $zip_file_path = $zip_dir_path . '/' . $zip_file_name; // مسیر فایل.
            $zip_file_url  = $zip_dir_url  . '/' . $zip_file_name; // URL.

            $zip = new ZipArchive(); // ایجاد ZIP.
            if ($zip->open($zip_file_path, ZipArchive::CREATE | ZipArchive::OVERWRITE) !== TRUE) { // باز کردن برای ایجاد/overwrite.
                wp_send_json_error(array('message' => 'نمی‌توان فایل ZIP ساخت. دسترسی نوشتن در پوشه uploads بررسی شود.')); // خطا.
            }

            $zip->setArchiveComment('LearnDash Assignments Bulk Download - Generated on ' . date('Y-m-d H:i:s')); // کامنت ZIP.
            $added_count = 0; // شمارنده اضافه‌شده.
            $errors      = array(); // آرایه خطاها.

            foreach ($ids as $assignment_id) { // حلقه روی IDها.
                if (get_post_type($assignment_id) !== 'sfwd-assignment') continue; // رد اگر نوع اشتباه.

                $file_path = get_post_meta($assignment_id, 'file_path', true); // مسیر از متا.
                if (empty($file_path) || !file_exists($file_path)) { // اگر خالی یا不存在.
                    $uploaded_files = get_post_meta($assignment_id, 'uploaded_files', true); // چک uploaded_files.
                    if ($uploaded_files && is_array($uploaded_files) && !empty($uploaded_files[0]['file_path'])) {
                        $file_path = $uploaded_files[0]['file_path']; // استفاده از اولین.
                    }
                }

                if (empty($file_path) || !file_exists($file_path)) { // fallback مسیرها.
                    $user_id  = get_post_field('post_author', $assignment_id); // ID کاربر.
                    $filename = get_post_meta($assignment_id, 'file_name', true); // نام فایل.
                    if (!$filename) continue; // رد اگر نام不存在.

                    $possible = array( // مسیرهای ممکن.
                        WP_CONTENT_DIR . "/uploads/learndash/assignments/{$user_id}/{$filename}",
                        WP_CONTENT_DIR . "/uploads/learndash/assignments/{$filename}",
                        $upload_dir['basedir'] . "/learndash/assignments/{$user_id}/{$filename}",
                        $upload_dir['basedir'] . "/learndash/assignments/{$filename}",
                    );
                    foreach ($possible as $path) {
                        if (file_exists($path)) { $file_path = $path; break; } // یافتن اولین موجود.
                    }
                }

                if (empty($file_path) || !is_file($file_path)) { // اگر هنوز یافت نشد.
                    $errors[] = "فایل یافت نشد: #{$assignment_id}"; // افزودن خطا.
                    continue;
                }

                $assignment = get_post($assignment_id); // پست تکلیف.
                $user       = get_userdata($assignment->post_author); // کاربر.
                $course     = get_post(get_post_meta($assignment_id, 'course_id', true)); // دوره.
                $lesson     = get_post(get_post_meta($assignment_id, 'lesson_id', true)); // درس.
                $topic      = get_post(get_post_meta($assignment_id, 'topic_id', true)); // موضوع.

                $prefix = sanitize_file_name( // پیشوند نام‌گذاری هوشمند.
                    ($user ? $user->display_name : 'Unknown') . '_' .
                    ($course ? $course->post_title : 'NoCourse') . '_' .
                    ($lesson ? $lesson->post_title : 'NoLesson') . '_' .
                    ($topic ? $topic->post_title : 'NoTopic') . '_ID' . $assignment_id
                );
                $new_name = $prefix . '.' . strtolower(pathinfo(basename($file_path), PATHINFO_EXTENSION)); // نام جدید با extension.
                $new_name = preg_replace('/[^A-Za-z0-9_\-\.\(\)]/', '_', $new_name); // پاکسازی کاراکترها.

                $stream = fopen($file_path, 'rb'); // باز کردن استریم برای خواندن.
                if ($stream && $zip->addFromString($new_name, stream_get_contents($stream))) { // افزودن به ZIP.
                    $added_count++; // افزایش شمارنده.
                } else {
                    $errors[] = "خطا در افزودن فایل: " . basename($file_path); // خطا.
                }
                if ($stream) fclose($stream); // بستن استریم.
            }

            $zip->close(); // بستن ZIP.
            $this->cleanup_zip_files(); // پاکسازی.

            if ($added_count > 0 && file_exists($zip_file_path)) { // اگر موفق.
                wp_send_json_success(array('url' => $zip_file_url)); // ارسال URL.
            } else {
                @unlink($zip_file_path); // حذف فایل ناموفق.
                $msg = 'هیچ فایلی اضافه نشد.'; // پیام پایه.
                if (!empty($errors)) $msg .= " | " . implode(' | ', array_slice($errors, 0, 5)); // افزودن ۵ خطا اول.
                wp_send_json_error(array('message' => $msg)); // ارسال خطا.
            }
        }

        /**
         * پاکسازی ZIPهای قدیمی‌تر از ۲۴ ساعت برای مدیریت فضای سرور و جلوگیری از انباشت فایل‌ها.
         * استفاده از glob برای لیست فایل‌ها و filemtime برای زمان.
         */
        public function cleanup_zip_files() {
            $upload_dir   = wp_upload_dir(); // دایرکتوری.
            $zip_dir_path = $upload_dir['basedir'] . '/ldam-zips'; // مسیر.
            if (!is_dir($zip_dir_path)) return; // خروج اگر不存在.

            $files = glob($zip_dir_path . '/*.zip'); // لیست ZIPها.
            $expiration = 24 * 60 * 60; // ۲۴ ساعت.
            foreach ($files as $file) { // حلقه.
                if (is_file($file) && (time() - filemtime($file) > $expiration)) { // اگر قدیمی.
                    @unlink($file); // حذف.
                }
            }
        }

        /**
         * اصلاح خودکار متادیتای lesson_id و topic_id هنگام آپلود تکلیف در موضوعات برای حل مشکلات LearnDash.
         * fallback برای استخراج lesson_id از parent، meta یا تابع learndash_get_lesson_id.
         * پارامترها: ID تکلیف، داده پست، ID صفحه فعلی.
         */
        public function fix_assignment_topic_metadata($assignment_id, $post_data, $current_page_id) {
            $current_page_id = intval($current_page_id); // پاکسازی.
            $page = get_post($current_page_id); // پست صفحه.
            if (!$page || $page->post_type !== 'sfwd-topic') return; // خروج اگر موضوع نباشد.

            $lesson_id = wp_get_post_parent_id($current_page_id); // parent به عنوان درس.
            if (!$lesson_id) $lesson_id = get_post_meta($current_page_id, 'lesson_id', true); // از متا.
            if (!$lesson_id && function_exists('learndash_get_lesson_id')) { // از تابع LearnDash.
                $lesson_id = learndash_get_lesson_id($current_page_id);
            }
            if ($lesson_id) { // اگر یافت شد.
                update_post_meta($assignment_id, 'lesson_id', $lesson_id); // ذخیره درس.
                update_post_meta($assignment_id, 'topic_id', $current_page_id); // ذخیره موضوع.
            }
        }
    }

    LearnDash_Assignments_Manager::get_instance(); // فعال‌سازی کلاس با Singleton برای شروع کار افزونه.
}
