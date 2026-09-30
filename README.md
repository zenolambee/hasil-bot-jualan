PHASE 30A: PASS

Audit Bot 1’s actual menu features, then reworked the UI without adding business features or changing order/payment/subscription logic.

Main menu is now arranged in two columns and links only to existing handlers.
Shop shows six products per page with working previous/next callbacks, popular and flash-sale shortcuts, and Main Menu navigation.
Product detail shows database-backed packages and prices. Availability uses package-specific plus generic inventory counts; INVITE and MANUAL packages show their existing delivery type.
Order detail and invite callbacks pass Telegram identity through ownership-checked APIs.
Verification: 180 tests passed, typecheck passed, lint passed, and builds passed for admin, backend, Bot 1, Bot 2, and worker.

Live Telegram smoke tests such as /start, button navigation, pagination, and payment callbacks were not run because a live bot session was not available.
