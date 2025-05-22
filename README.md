# mcc
Forum 
# Odoo v18 custom module: Forum Access via Membership Tiers + Website Pages

from odoo import models, fields, api

class MembershipProduct(models.Model):
    _inherit = 'product.template'

    is_membership = fields.Boolean(string="Is a Membership")
    membership_tier = fields.Selection([
        ('bronze', 'Bronze'),
        ('silver', 'Silver'),
        ('gold', 'Gold'),
    ], string="Membership Tier")


class ResPartner(models.Model):
    _inherit = 'res.partner'

    membership_tier = fields.Selection(related='membership_line_ids.membership_id.membership_tier', string="Membership Tier", store=True)


class WebsiteForum(models.Model):
    _inherit = 'forum.post'

    @api.model
    def create(self, vals):
        user = self.env.user
        if not user.has_group('base.group_portal'):
            raise AccessError("Only logged-in members can post.")

        tier = user.partner_id.membership_tier
        if tier != 'gold':
            raise AccessError("Only Gold members can post in the forum.")
        return super().create(vals)


# Access Control via controllers
from odoo import http
from odoo.http import request

class ChamberPortal(http.Controller):

    @http.route(['/chamber/newsletters'], type='http', auth='user', website=True)
    def newsletters(self, **kw):
        tier = request.env.user.partner_id.membership_tier
        if not tier:
            return request.render('website.403')
        return request.render('chamber_portal.newsletters')

    @http.route(['/chamber/members'], type='http', auth='user', website=True)
    def members(self, **kw):
        tier = request.env.user.partner_id.membership_tier
        if tier not in ['silver', 'gold']:
            return request.render('website.403')
        members = request.env['res.partner'].search([('membership_tier', '!=', False)])
        return request.render('chamber_portal.members', {'members': members})

    @http.route(['/chamber/community'], type='http', auth='user', website=True)
    def community(self, **kw):
        tier = request.env.user.partner_id.membership_tier
        if tier != 'gold':
            return request.render('website.403')
        posts = request.env['forum.post'].search([])
        return request.render('chamber_portal.community', {'posts': posts})


# XML Website Templates (to be placed in /views/templates.xml)

"""
<odoo>
    <template id="chamber_portal.newsletters" name="Newsletters">
        <t t-call="website.layout">
            <div class="container">
                <h2>Newsletters</h2>
                <p>Welcome to the members-only newsletter section.</p>
            </div>
        </t>
    </template>

    <template id="chamber_portal.members" name="Members Directory">
        <t t-call="website.layout">
            <div class="container">
                <h2>Member Directory</h2>
                <t t-foreach="members" t-as="member">
                    <p><t t-esc="member.name" /> - <t t-esc="member.email" /></p>
                </t>
            </div>
        </t>
    </template>

    <template id="chamber_portal.community" name="Community Forum">
        <t t-call="website.layout">
            <div class="container">
                <h2>Community Forum</h2>
                <t t-foreach="posts" t-as="post">
                    <h4><t t-esc="post.name" /></h4>
                    <p><t t-esc="post.body" /></p>
                </t>
            </div>
        </t>
    </template>
</odoo>
"""

# Membership can be tied to a sale order confirmation trigger or payment event to grant access

# Access control, security rules, and menu entries must be configured in corresponding XML files
